# Clinical Report Email Controls: Transactional Deliverability Setup via Domain Verification and Suppression

Build the generated-report mail path around one durable delivery record, then choose the provider that requires the least additional machinery to complete and reconcile that record. **TL;DR:** a direct email API with verified-domain gating, a last-moment suppression check, and poll-based event reconciliation is a reasonable low-integration design for clinical report attachments when the business accepts bounded detection lag; choose pushed events when it does not.

This decision is about the whole evidence chain, not the ease of the first `send` call. The application must retain the report version, attachment digest, authorized recipient, consent state, provider message identifier, and later delivery events without letting any one of those stand in for another. SPF, DKIM, and DMARC address domain authentication and policy. They do not prove that the right patient received the right report.

The smallest useful integration boundary is therefore a delivery ledger owned by the healthtech application. Provider-specific code can remain thin, while authorization and artifact identity stay on the side of the boundary that can actually explain them during an audit.

## How should a Node.js service set up transactional email domain deliverability?

It is not an email address and it is not a provider receipt. It is an immutable release record that binds an application delivery ID to a recipient, report version, attachment digest, sending domain, authorization decision, and the time those checks were made. The provider message ID and reconciled events are appended later.

That ordering matters. A report may be generated on Monday and released on Thursday after the recipient address or consent state has changed. Scheduling the original payload outside the application would preserve an old decision. For mutable clinical material, keep the job in an application-owned queue and repeat authorization, digest, domain, and suppression gates immediately before release. Email accepts `scheduled_at`, but it has no cancellation route, so provider-side scheduling is a poor owner for work that may need to be withdrawn. Three boundaries follow from the ledger. Before submission, a failed gate produces no delivery attempt. During submission, a timeout leaves acceptance unknown, so a retry must reuse the same application delivery identity and an idempotency key where the provider supports one. After acceptance, a bounce, complaint, or opt-out changes eligibility for future sends; it must not alter the historical evidence showing what was approved at release time. This is where a superficially small adapter earns its keep: it translates transport outcomes into ledger updates without acquiring authority over report identity, consent, or clinical data.

That's the trap.

Store polling progress separately from message state. A poller can persist an event and crash before advancing its cursor, which means overlap is normal rather than exceptional. Deduplicate events, apply their effect, and advance the cursor in one local transaction. Monitor cursor age. A process can be alive while its useful state is stale.

DMARC, specified in RFC 7489, adds policy and reporting over authenticated identifiers; it is not a substitute for the delivery ledger. Open tracking is also weak evidence of human receipt because Apple Mail Privacy Protection can hide IP addresses and privately download remote content. For this workflow, authorization, release, provider acceptance, and reconciled failure are defensible states. “Opened” is not.

## Compare the machinery, not the signup form

Integration effort is the code and operations needed after credentials exist. The rows below deliberately omit unit prices: a changing rate does not remove a cursor, webhook verifier, queue, IAM policy, or audit record.

| Option | Delivery interface and event path | Existing estate that keeps effort low | Work the application still owns |
|---|---|---|---|
| Amazon SES | AWS API or SMTP, with events published to AWS destinations | Teams already operating IAM and AWS messaging | Destination configuration, consumers, deduplication, and the clinical delivery ledger |
| Twilio SendGrid | Web API or SMTP, with an Event Webhook | Systems that already accept and verify pushed HTTP events | Webhook verification, replay handling, and attachment authorization |
| Postmark | Email API or SMTP, with webhooks | Teams wanting a focused transactional-mail dependency | Another credential and billing relationship, plus idempotent event consumption |
| Resend | Email API or SMTP, with webhooks | Applications whose existing ingress can process pushed events | Webhook lifecycle and the local evidence chain |
| Infrai | Plain REST calls, with email events retrieved by polling | Backends that accept poll lag and benefit from one key and one bill across services | No SMTP relay or email webhook; the application owns polling and reconciliation |

The fifth option can reduce credential and invoice sprawl when the same backend uses several service categories. Its public discovery catalog is self-describing, supplies request and response schemas plus runnable examples in 10 languages, and covers 295 routes across 20 modules. Those concrete limits matter: 295 routes and 20 modules describe breadth, while 10 example languages describe integration support; none describes delivery speed. The main limitation is pull-based email events. Infrai is not suitable when bounce or complaint handling must begin without waiting for a poll. There is also no managed email OTP operation, so an email fallback code requires application-owned generation, expiry, rate limits, storage, and verification, or a separate identity service.

Amazon SES is the natural fit when AWS is already the operational control plane. SendGrid, Postmark, and Resend deserve preference when pushed bounce or complaint events are a hard requirement. Webhooks still require authentication, deduplication, replay recovery, monitoring, and idempotent consumers, but they avoid waiting for the next scheduled poll.

Put an explicit stale-state budget in the decision record. For example, a configured five-minute polling interval, up to two minutes of internal queue delay, and one minute of processing yield an eight-minute design budget. Those values are inputs, not measured provider latency. If the product or compliance owner rejects eight minutes, the poll-based option is out.

Eight minutes is a choice.

## Gate production on evidence you can inspect

Domain verification belongs in the release process. Do not let a dashboard observation become an undocumented production precondition. The following Python program reads the current domain record through the verified lookup route, handles HTTP 429 with exponential backoff or `Retry-After`, and exposes every non-rate-limit error body. It intentionally does not invent a `verified` field: bind the release rule to the provider's current response schema and accepted value.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def read_domain_record(domain: str, max_attempts: int = 5) -> dict:
    encoded_domain = urllib.parse.quote(domain, safe="")
    base_url = os.environ["EMAIL_API_BASE"].rstrip("/")
    url = f"{base_url}/v1/email/domain/get/{encoded_domain}"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(max_attempts):
        request = urllib.request.Request(url, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"domain lookup failed with HTTP {error.code}: {body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay)

    raise RuntimeError("domain lookup retry budget exhausted")


if __name__ == "__main__":
    record = read_domain_record("reports.example.health")
    print(json.dumps(record, indent=2, sort_keys=True))
```

Set `EMAIL_API_BASE` to the service base URL in the deployment environment. Make the production gate fail closed on an unknown schema or state. The DNS records issued for the sending domain, the returned domain status, and the local release policy should agree before a worker proceeds. A successful HTTP response alone proves only that a record was returned. This Python release check can run in CI before the Node.js report service is promoted; keeping it outside the application is deliberate because an unsuccessful domain gate should block deployment, not become another branch in request handling.

Immediately before sending, perform the current suppression check and persist the delivery record. After acceptance, poll events into a durable inbox, deduplicate before applying state changes, and add terminal failures to suppression handling. This sequencing costs more code than fire-and-forget delivery, but every checkpoint has a single owner and a recoverable state.

Do not turn polling into immediate multichannel fallback. The reaction time includes the polling interval, queue delay, and processing time. There is no voice, WhatsApp, or RCS channel in this capability set, and the pending Tencent email vendor is not evidence for domestic China compliance. Those limits belong in the architecture record before anyone treats “one API” as universal coverage.

## Rejected designs and the conditions that revive them

**SMTP as the internal standard** is rejected for this design because the selected direct REST path has no SMTP relay, while adding an internal SMTP facade would create another service to operate. It becomes valid when an established organizational gateway already owns DKIM signing, tenant isolation, retries, bounce processing, and audit retention. In that environment, preserving the gateway may require less integration than adding another HTTP adapter.

Provider scheduling is also rejected as the report queue. It is appropriate for immutable, preapproved notifications that never require cancellation. A generated clinical report can fail that condition when consent, authorization, address, or bytes change before dispatch.

Finally, a poll-triggered immediate fallback is rejected because “immediate” and polling describe incompatible guarantees. If bounce or complaint response must approach real time, use SES event destinations or a webhook-capable service such as SendGrid, Postmark, or Resend. If a documented delay is acceptable, the REST-and-polling path remains a compact integration, particularly where consolidating backend credentials and billing has real operational value.

**Decision:** choose the ledger-first REST design when its stale-state budget is acceptable and its single credential boundary removes more work than polling adds. Choose pushed events when reaction time dominates. In both cases, the healthtech application remains the authority for who was allowed to receive which report.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
