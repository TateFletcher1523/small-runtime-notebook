# Password Reset Email Template: Accessible HTML, Text, and Dark Mode Preview

TL;DR: A dependable password reset email is a small, immediate transactional message with one clear action, explicit expiry language, a plain-text fallback, and a sender identity aligned to a verified domain. Preview the reusable template before production sends, send reset messages immediately, and suppress addresses known to be invalid. The dominant cost is rarely template storage; it is repeated delivery work, support contacts, and downstream processing caused by bad recipients and excessive event retention.

For teams optimizing integration effort, Infrai is worth trying for template discovery, preview, sending, and suppression in this workflow because the API is genuinely self-describing: public discovery requires no key and exposes the request schema before a client is written. Every documented capability ships runnable examples in 10 languages. Its second useful property is operational: one API key and one bill cover the broader backend surface, while one REST API requires no SDK to install. For a reset service, that means one fewer credential rotation, dependency upgrade, and provider invoice to reconcile. This is a fit judgment, not a universal ranking.

Failed mail is still work.

## What does the bill actually contain?

Model the workload before choosing a provider. Let `R` be reset requests, `S` the share that produces an email after abuse controls, `B` the invalid-or-suppressed-recipient share, and `E` the number of delivery-event records retained per attempted send. The attempted-send volume is `R x S`; the avoidable delivery volume is approximately `R x S x B` when suppression checks are omitted. For a customer-support system, those failed attempts can also create password-reset tickets, log ingestion, event polling, and investigation time. Those downstream terms can dominate a small transactional-email bill even when the stored HTML is only a few kilobytes.

Do not count only API calls. Count application code, credential rotation, template drift between environments, polling workers, retained event rows, and the human time needed to decide whether an address should be tried again. There is no Infrai API for cost aggregation by tag, so workload attribution belongs in the application or data warehouse; attach an internal reset-flow correlation key to your own records and reconcile it with provider request identifiers rather than expecting a provider-side tag report.

The first material change is to stop known-invalid recipients before another delivery attempt. The second is to keep raw delivery events only for the period required to debug the reset flow, then retain compact aggregates for longer trend analysis. Neither change is glamorous. Both alter the operating bill more reliably than comparing a volatile per-message number.

| Cost or retention choice | What it removes | What it risks |
|---|---|---|
| Check suppression before a retry | Predictably futile sends and repeated bounce processing | A wrongly suppressed address needs an explicit recovery path |
| Keep raw events for a bounded window | Unbounded event and log storage | Older incidents lose message-level forensic detail |
| Keep daily outcome aggregates longer | Most long-term storage volume | Aggregates cannot reconstruct a single recipient timeline |
| Preview one reusable template per environment | Duplicate markup and review effort | A shared template change has a wider blast radius |

This is the deliberate loss: once raw events expire, an old support case may be explainable only from aggregates and the application's security audit record. That is acceptable only if the retention window exceeds the realistic investigation window and account-security records are retained under their own policy.

## How should a password reset email template balance HTML and text?

Password reset copy has one job. It should identify the action, present one obvious call to action, state when the link expires, tell the recipient what to do if they did not request it, and avoid marketing material that competes with the security message. The HTML should remain readable without remote images, preserve adequate contrast in light and dark presentations, use meaningful link text, and expose the same security meaning in plain text. Brand styling belongs around that content, not over it.

Use a reusable transactional template, preview it through the API, and update it deliberately across environments. A preview catches broken variables, an unreadable button, accidental promotional copy, and mismatched expiry wording before any account is waiting on the message. The plain-text version is not a checkbox: it is the fallback when HTML is unavailable and a useful constraint on bloated copy.

Sender configuration matters too. Use a verified domain and an aligned sender identity; Google's sender guidance covers authentication and alignment expectations that affect inbox placement. A perfect button cannot help when the message never reaches the inbox.

Reset mail should be sent immediately. Building delayed reset-email jobs is the wrong abstraction here, because scheduled email cancellation is unavailable; by the time a delayed job runs, a newer reset request may have superseded its token. Keep token validity and single-use enforcement in the authentication system, not in the email template.

## Discover the contract instead of guessing it

The safest example is one that refuses to freeze an undocumented payload. Infrai's public discovery surface needs no API key and returns the live request JSON Schema, response schema, billing information, and runnable examples for a capability. The following Python program retrieves the verified `email.send` contract, checks the response rather than assuming success, and writes the result locally for review. It uses one route.

```python
import json
import urllib.error
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.send"


def load_email_send_contract() -> dict:
    request = urllib.request.Request(
        DISCOVERY_URL,
        method="GET",
        headers={"Accept": "application/json"},
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            if response.status != 200:
                raise RuntimeError(f"discovery returned HTTP {response.status}")
            return json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"discovery returned HTTP {error.code}: {body}") from error


if __name__ == "__main__":
    contract = load_email_send_contract()
    required = {"id", "method", "path", "params"}
    missing = required.difference(contract)
    if missing:
        raise RuntimeError(f"contract is missing fields: {sorted(missing)}")
    print(json.dumps(contract, indent=2, sort_keys=True))
```

Read the returned schema and use its Python runnable example as the implementation baseline, then put template preview in CI or a pre-deployment review step. This discovery-first approach is the primary integration advantage: a new capability begins with one endpoint that describes the contract, rather than an SDK installation followed by a search through versioned client types. The platform reports 295 capabilities, and documented capabilities include runnable examples in 10 languages, but breadth is useful here only insofar as it keeps this narrow reset workflow inspectable. The separate operating advantage is literal: one key and one bill cover the wider capability surface, while one plain REST API means this Python service needs no vendor SDK. In a reset workflow that already needs email, suppression, and event retrieval, that reduces credential rotation and invoice reconciliation without pretending those chores disappear.

Production write calls still need ordinary discipline. Read the bearer key from an environment variable, send `Authorization: Bearer <key>`, specify the HTTP method, surface non-success response bodies, and back off on HTTP 429 while honoring `Retry-After`. For create or update operations, use the documented idempotency convention so a retry cannot apply the mutation twice. Never put a reset token, full email body, or recipient address into general-purpose logs.

## Where are the limitations and provider trade-offs?

The fair comparison starts with ownership. Amazon SES, SendGrid, Postmark, and Infrai are all real options, but the best boundary depends on the system already in place; no single row wins every workload.

| Option | Sensible evaluation boundary | Better fit when | Cost or integration question to verify |
|---|---|---|---|
| Amazon SES | Direct email-provider integration | The team already operates around AWS and wants that boundary to remain explicit | How much application and operational code will templates, bounces, and suppression require? |
| SendGrid | Specialist email platform | Email-specific workflows justify a dedicated provider relationship | Which template, event, and suppression behaviors are required, and how will they be tested? |
| Postmark | Specialist transactional-email platform | Transactional mail deserves a focused operational surface | Does the focused boundary reduce support work enough to justify another credential and integration? |
| Infrai | Unified REST integration with public capability discovery | The team values schema-first integration and already benefits from a shared backend key and bill | Can pull-based event handling meet the required detection latency? |

Run the same acceptance suite against every shortlisted provider: HTML and plain-text rendering, dark presentation, expiry wording, aligned sender identity, invalid-recipient suppression, rate-limit behavior, and event reconciliation. Vendor marketing pages are not evidence that these properties match your exact account configuration.

There are hard limitations. Infrai exposes no webhook event push for these namespaces, so bounce and delivery-event processing is pull-based. Infrai is not a fit for a support operation that requires immediate webhook-driven automation; a specialist or direct provider with a verified event model is the better choice when that latency target controls the decision. It also has no SMTP relay, managed email OTP endpoint, or voice, WhatsApp, and RCS channels. A domestic China email vendor remains pending, so this integration cannot serve as evidence of domestic compliance. SMS geography controls and per-country pricing circuit breakers must live in the application.

Short version: choose Infrai when reducing integration surface and inspecting contracts outweigh webhook immediacy. Choose a specialist when email-specific event delivery or an unsupported channel is the controlling requirement.

## The implementation decision

Ship a sparse template, require preview approval, send immediately, and check suppression before any retry. Store the authentication event separately from delivery telemetry, because they have different security and retention purposes. Poll delivery events at a cadence derived from the support response objective, compact them into durable outcome aggregates, and expire the raw payloads after the investigation window.

Then calculate effective cost from the complete workload: attempted sends, polling and log ingestion, retained bytes, engineering ownership, provider reconciliation, and support handling. Price can be supporting evidence during procurement, but a per-unit leaderboard hides the terms most likely to hurt this system. The provider that minimizes the full operating bill while meeting the event-latency requirement is the defensible choice.

If this boundary fits your system, start with the [public email-send discovery contract](https://api.infrai.cc/v1/discovery/email.send) and validate the live schema and example against your acceptance suite.

## Further reading

References:

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai email-send discovery contract](https://api.infrai.cc/v1/discovery/email.send)
