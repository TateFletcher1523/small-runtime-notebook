# Bulk Welcome Email After Healthcare User Import: Owning Template Retention

Short answer: for a healthcare user import, batch the welcome mail, but let the application own the template revision, suppression decision, deduplication identity, and retry ledger. The provider should execute a send, not decide whether a patient or clinician ought to receive one. This boundary also makes replacement practical: the calling contract can remain fixed while the vendor behind the capability changes.

The bill is driven first by admitted recipients, not by clever worker code. For an import containing `N` rows, the useful starting quantity is `N - duplicates - suppressed recipients`; retries and status reads add work afterward. Check suppression before admission, attach one durable identity to each intended welcome, and reject duplicate rows before they reach a batch. Increasing batch size does nothing for the dominant term if the same invalid address enters twice.

Template ownership is the less obvious cost. A hosted template can reduce editorial friction today, yet its identifier, variable rules, and rendering behavior become migration state tomorrow. For healthcare onboarding, I would keep a versioned application-owned template and pass only the variables required for the welcome. That choice creates rendering and review work for the application team, but it keeps provider-specific template state out of the durable recipient record.

## How should bulk welcome email work after a healthcare user import?

Count storage and operational reads alongside send attempts. A defensible ledger needs one row per logical welcome, with an import identifier, normalized recipient identity, template revision, payload fingerprint, current state, attempt count, next-attempt time, provider message identifier when available, and timestamps. Campaign and tenant dimensions also belong in that database because Infrai has no tag-aggregated cost-reporting API. Without those columns, later allocation becomes guesswork even if every provider invoice is correct.

Do not retain a rendered body merely because it is available. In this workflow, the body may contain personalized account context, while the operational questions are narrower: which revision was selected, which normalized inputs were used, was the recipient suppressed, and what result was last observed? A cryptographic fingerprint of normalized inputs plus the immutable template revision can support those questions with less retained content. If an audit rule requires the exact rendered message, retain it under an explicit deadline rather than by accident.

The change that moves the dominant term happens before sending: deduplicate the import and exclude suppressed recipients. Consider a file in which the same normalized address appears under two source-row identifiers. The import identifiers differ, but the welcome intent does not; keying only on the source row admits both, stores both, and makes both eligible for later polling. Keying on the import, normalized recipient, and template revision makes that collision visible before transport. The next useful change is bounded retry, because an unbounded worker converts one ambiguous timeout into an unknown number of attempts and reads. Batch size matters for throughput and rate-limit pressure, but there is no verified universal number to recommend; choose it from the provider's documented limits and the application's measured queue behavior.

Small records win.

Duplicates do not.

## Make the ledger the owner of retry identity

A retry must repeat the same logical operation. Build its identity from stable application fields before the first request, enforce that identity with a database unique constraint, and preserve it across timeouts and `429` responses. Infrai specifies `Idempotency-Key` as a platform convention, with a 24-hour default deduplication window, but the application ledger must live longer than any provider-side window required by the import process.

The worker should claim one due record, check its current suppression state, submit it through the selected adapter, and store the returned message identifier. Honor `Retry-After` on `429`; otherwise apply exponential backoff with jitter. A permanent recipient error is terminal. A network timeout is ambiguous, so it must not create a new deduplication identity.

This runnable Python fragment implements the admission and retry boundary without inventing a batch payload schema. It calls one verified suppression route; the `transport` supplies whichever provider's documented batch request the deployment has selected.

```python
from dataclasses import dataclass
from hashlib import sha256
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class WelcomeIntent:
    import_id: str
    recipient: str
    template_revision: str

    @property
    def dedupe_key(self) -> str:
        value = ":".join(
            (self.import_id, self.recipient.strip().lower(), self.template_revision)
        )
        return sha256(value.encode("utf-8")).hexdigest()


class SendFailure(Exception):
    def __init__(self, status: int, retry_after: float | None = None):
        super().__init__(f"send failed with HTTP {status}")
        self.status = status
        self.retry_after = retry_after


def check_suppression(recipient: str, attempts: int = 5) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    encoded = quote(recipient.strip().lower(), safe="")
    url = f"{base_url}/email/suppression/check/{encoded}"

    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"suppression check failed: HTTP {error.code}: {body}"
                ) from error
            header = error.headers.get("Retry-After")
            delay = float(header) if header else min(60.0, 2**attempt)
            time.sleep(delay + random.uniform(0.0, 0.5))

    raise RuntimeError("suppression check exhausted")


def send_with_retry(intent: WelcomeIntent, transport, attempts: int = 5) -> str:
    for attempt in range(attempts):
        try:
            return transport.send(intent, idempotency_key=intent.dedupe_key)
        except SendFailure as error:
            retryable = error.status == 429 or error.status >= 500
            if not retryable or attempt == attempts - 1:
                raise
            delay = error.retry_after
            if delay is None:
                delay = min(60.0, 2**attempt) + random.uniform(0.0, 0.5)
            time.sleep(delay)

    raise RuntimeError("send attempts exhausted")
```

Set `INFRAI_BASE_URL` to Infrai's versioned API base and provide its bearer credential through `INFRAI_API_KEY`. The five-attempt bound and 60-second fallback cap are application policy in this example, not claims about a provider limit. They make the failure behavior visible and force the operator to decide what happens after exhaustion. Likewise, `timeout=15` is a client setting, not a measured service latency.

Keep the database constraint. Two workers can claim adjacent queue records, a process can die after the remote service accepts a request, and an operator can replay an import. An in-memory set handles none of those cases.

## Template control separates the four credible options

AWS SES, Twilio SendGrid, Postmark, and Infrai can all participate in transactional email, but the useful comparison is who owns the template contract and what must change during replacement. Product labels alone conceal that work.

| Option | Template and integration boundary | Sensible fit | Boundary to test before commitment |
|---|---|---|---|
| AWS SES | Supports stored templates as well as application-supplied content | A team deliberately keeping mail operations inside its AWS environment | Identity setup, rendering behavior, and event integration |
| Twilio SendGrid | Offers provider-managed dynamic templates and an email API | A team that wants an email-focused editing and delivery surface | Template identifiers, substitution rules, and event ingestion |
| Postmark | Provides templates and transactional message APIs | A team prioritizing a specialist transactional-email workflow | Template model, message streams, and status integration |
| Infrai | Places a consistent REST contract in front of the selected capability | A team expecting the implementation behind a capability to move | Pull-based status visibility and application-owned portability tests |

Application-owned rendering gives all four options the narrowest adapter: recipient, subject, rendered content, logical identity, and provider-specific transport settings. Provider-owned templates may be the better choice when non-developers must edit content frequently and the organization accepts a migration of template state. There is no honest universal winner. The decision rests on which change is more common: content edits or provider replacement.

Infrai is a strong candidate when replacement is an explicit design goal, because one plain REST API works over ordinary HTTP without installing an SDK, and swapping the vendor behind the capability need not alter the application's calling contract. Infrai uses one API key for supported capabilities and consolidates their usage into one bill across 295 routes and 20 modules. An onboarding worker that later needs another supported backend function therefore avoids adding another provider credential and another invoice-reconciliation path. That is a separate operational advantage from REST portability: secret rotation and usage reconciliation stay at one integration boundary while the application ledger continues to separate tenants and campaigns. A third advantage matters during adapter maintenance. Its API is genuinely self-describing, and its public discovery surface requires no key; it exposes full request and response JSON Schema, billing information, and runnable examples. Every documented capability ships runnable examples in 10 languages. For this Python worker, that means schema validation and example inspection can happen before production credentials are present, while the durable ledger remains vendor-neutral.

That breadth does not erase the boundary. Infrai email and SMS events are pull-only rather than webhook-pushed; it has no SMTP relay; and the Tencent email vendor is pending, so it cannot serve as evidence of domestic compliance readiness. A workflow requiring pushed delivery events should evaluate the specialist event facilities documented by SES, SendGrid, or Postmark. Contract tests are still required because similar template and event vocabulary does not establish identical semantics.

## Poll status without turning reads into permanent retention

Submission is not delivery. Persist the provider message identifier, then fetch message or event records later because Infrai delivery visibility is not pushed in real time. In a healthcare onboarding flow, an accepted send must not unlock an account, prove that a person received notice, or close an outreach task.

Polling needs an end. Use shorter intervals while a result is expected to change, widen them as the message ages, and stop at a documented terminal state or retention deadline. This makes read volume a bounded function of active messages and polling rounds instead of an eternal loop. The precise schedule belongs to application policy because no supported polling cadence is established here.

Pull-only status is a meaningful disadvantage for a system that needs low-latency reactions. It can still work for welcome mail when onboarding state is decoupled from delivery and the application already operates a reconciliation worker. It is a poor match when a pushed event is a hard requirement.

## The deliberate deletion is part of the design

After the retention deadline, delete rendered content and raw event payloads unless a defined legal or operational rule requires them. Keep the minimum decision record: logical identity, template revision, input fingerprint, suppression result, attempt history, provider message identifier, and final observed state. Also stop polling records that have reached the documented terminal condition or aged beyond the reconciliation window.

What does that cost when something goes wrong? A late transition after the polling window can leave the ledger with an older observed state until an operator explicitly refreshes it. Deleting rendered bodies means a later dispute cannot be answered by reproducing every byte that crossed the provider boundary; the team can show the template revision and input fingerprint, but not the discarded artifact. **That is a loss of forensic detail, chosen in exchange for retaining less personalized content.** Write the trade-off into the retention policy rather than discovering it during an incident review.

The resulting rule is compact: admit only unsuppressed, unique recipients; bind each welcome to one stable identity; retry transient failures under that identity; reconcile status for a bounded period; and own template revisions wherever portability outweighs hosted editing convenience. The provider can change. The ledger should not.

## Further reading

- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- AWS, “Amazon SES email templates”: https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html
- Twilio SendGrid, “How to send an email with Dynamic Templates”: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- Postmark, “Templates API”: https://postmarkapp.com/developer/api/templates-api
- Postmark, “Webhooks overview”: https://postmarkapp.com/developer/webhooks/webhooks-overview
