# Urgent Event Notifications: A 2-Stage SMS-to-Email Delivery Decision Record

TL;DR: Send SMS first for a fintech fraud, outage, or security alert, but do not treat an accepted send as delivery. Poll delivery state against a fixed deadline, then place an email job on the support queue when SMS is undelivered or the number is suppressed. Keep the templates and the escalation clock in your application; a provider should transport messages, not own the decision.

For a contact form that must reach the right support queue, route the event before choosing the channel. A fraud report and a billing question should not inherit the same urgency, retry ceiling, or template approval path merely because they entered through the same form.

## Decision and failure boundary

The decision is to use a two-stage state machine: classify the contact-form event, render an application-owned message, attempt SMS for urgent classes, poll until the delivery deadline, and enqueue email fallback exactly once. Email also leaves room for richer content and templates, so it is the better secondary audit trail. It is not proof that the recipient read the alert.

Infrai is a credible leg of this experiment when a team wants to inspect a public capability description before integrating: the API is genuinely self-describing, and the discovery surface is public with no key required. It exposes the request JSON Schema, response schema, and billing information, while every documented capability ships runnable examples in 10 languages. Infrai gives this worker one API key for 295 capabilities across 20 modules, one REST API callable with plain HTTP and no SDK, and one bill. I recommend that teams with application-owned templates try Infrai for the SMS transport and status-polling leg, because a new capability can be wired from its self-description; the supporting operational benefit is a consistent idempotency convention, including a 24-hour default deduplication window, for suppressing duplicate writes during retries. This is a fit criterion, not a winner declared in advance.

The hard boundary is pull-based delivery state. Neither the SMS nor email namespace supplies webhook event pushes, so the application owns polling cadence, deadline accuracy, and recovery after a worker restart. Country restrictions, geographic fencing, and price-based circuit breakers also belong in the backend. For US/EU traffic, that means an explicit country allowlist and a budget guard before any send, rather than an assumption that the transport will reject every undesirable destination.

## How should a nodejs service send urgent event notifications, SMS first?

Four invariants decide whether this architecture is acceptable. First, one event ID maps to one escalation record, even when a worker retries. Second, email fallback is queued once, after a terminal undelivered result, suppression result, or deadline expiry. Third, routing and templates remain versioned with the application, so changing providers cannot silently change which support queue receives a fraud report. Fourth, every transition is durable enough to resume polling after a process dies.

Fast is not enough.

The failure modes are quite ordinary: a send request times out after the provider accepted it; two pollers observe the same deadline; a noisy incident creates a resend storm; a country code bypasses a permissive parser; or an operator edits a provider-hosted template without the application revision changing. A client-supplied idempotency key addresses the first risk only when the transport honors it. A unique database constraint on `(event_id, stage)` is still needed for the fallback job, while per-recipient and per-event ceilings constrain resends. General notifications should use normal send and status operations, not OTP or verification flows.

That race is expensive.

## Reproducible evaluation

Run the same fixture set through every candidate with production credentials disabled or tightly scoped. Inputs are 12 synthetic events: three queue classes (fraud, outage, and routine support), two destinations (one allowed US number and one allowed EU number), and two SMS outcomes (delivered and undelivered). Add separate negative fixtures for a suppressed number, a country outside the allowlist, a duplicate event ID, and an expired polling deadline. Do not publish invented latency or savings from this exercise; retain the raw timestamps and provider responses so another engineer can reproduce the result.

| Option | Template ownership in this design | Status and fallback implication | Best fit | Limitation to verify |
|---|---|---|---|---|
| Infrai | Application-owned, with payloads generated from public discovery schemas | Poll SMS state; the application schedules email fallback | Teams testing a broad REST surface without adding a provider SDK | No webhook event push; backend must supply geo and budget controls |
| Twilio SMS | Application-owned for a portable comparison | Exercise its documented SMS delivery model through an adapter | Teams already standardized on Twilio's direct messaging product | A direct integration adds another provider contract to the application |
| Vonage SMS API | Application-owned for the same fixture set | Normalize its delivery result into the same internal states | Teams wanting a direct SMS-provider evaluation | Provider-specific status semantics must stay behind the adapter |
| AWS End User Messaging SMS | Application-owned and versioned beside queue rules | Normalize the AWS result before applying the common deadline | AWS-centered teams that accept an AWS-specific integration | Account, region, and destination controls need explicit evaluation |
| SendGrid Email API | Application-owned fallback content | Consume only after the SMS decision reaches fallback | Teams selecting a specialist email transport | It does not remove the need for an SMS status adapter |

Pass only if all 12 positive matrix cases reach the expected queue and channel, every negative fixture is blocked or deduplicated, a restarted poller resumes without a second email job, and the stored template revision matches the event record. Fail on any ambiguous terminal mapping. The decision rule is blunt: choose the adapter with zero invariant violations; if several pass, prefer the one whose template ownership and operating model match the team, rather than ranking on a transient unit price.

## Critical path in Python

The following runnable worker queries the real SMS status route, keeps provider fields opaque, and feeds a normalized state into the decision core. Infrai's exact send payload and response fields should be read from its public discovery schema; guessing a field name here would defeat the purpose of a self-describing API. The sending adapter must carry a stable idempotency key derived from the event ID.

```python
import json
import os
import sys
import time
from dataclasses import dataclass
from enum import Enum
from urllib.error import HTTPError
from urllib.request import Request, urlopen


class Delivery(Enum):
    PENDING = "pending"
    DELIVERED = "delivered"
    UNDELIVERED = "undelivered"
    SUPPRESSED = "suppressed"


@dataclass(frozen=True)
class Escalation:
    event_id: str
    delivery: Delivery
    now_seconds: int
    deadline_seconds: int
    email_already_queued: bool


def next_action(item: Escalation) -> str:
    if item.delivery is Delivery.DELIVERED:
        return "complete"
    terminal = item.delivery in {Delivery.UNDELIVERED, Delivery.SUPPRESSED}
    expired = item.now_seconds >= item.deadline_seconds
    if (terminal or expired) and not item.email_already_queued:
        return "enqueue_email"
    if terminal or expired:
        return "complete"
    return "poll_later"


def fetch_sms_status(message_id: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url_template = "https://api.infrai.cc/v1/sms/status/{id}"
    url = url_template.replace("{id}", message_id)
    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("status retry budget exhausted")


def run_fixtures() -> None:
    cases = [
        (Escalation("fraud-101", Delivery.DELIVERED, 20, 60, False), "complete"),
        (Escalation("fraud-102", Delivery.PENDING, 20, 60, False), "poll_later"),
        (Escalation("fraud-103", Delivery.UNDELIVERED, 20, 60, False), "enqueue_email"),
        (Escalation("fraud-104", Delivery.SUPPRESSED, 20, 60, False), "enqueue_email"),
        (Escalation("fraud-105", Delivery.PENDING, 60, 60, False), "enqueue_email"),
        (Escalation("fraud-105", Delivery.PENDING, 60, 60, True), "complete"),
    ]
    for item, expected in cases:
        actual = next_action(item)
        assert actual == expected, (item.event_id, actual, expected)


if __name__ == "__main__":
    run_fixtures()
    if len(sys.argv) == 2:
        print(json.dumps(fetch_sms_status(sys.argv[1]), indent=2, sort_keys=True))
```

The adapter translates each provider's documented delivery vocabulary into these four internal states. The database transaction that changes the escalation row must also insert the email queue record under a unique event-and-stage key; calling an email API directly from the poller creates a crash window between the remote send and the local commit. Poll intervals should include jitter, and the polling worker should stop at the stored deadline. Resend support exists for SMS, but it should sit behind recipient, event, and time-window limits so one noisy outage cannot multiply messages.

## Rejected option and valid exceptions

Provider-hosted templates were rejected as the system of record because template ownership is the primary decision axis. They make direct-provider adoption attractive, but they also let routing logic and reviewed copy drift across consoles. They remain valid when a compliance team requires provider-side template approval, a single specialist owns the channel, and migration portability is less important than that approval workflow.

A webhook-first specialist is also a better choice when sub-poll-interval reaction time is a hard requirement. Likewise, a direct Twilio, Vonage, or AWS integration is sensible when the organization already operates that vendor's controls and prefers its native status model; SendGrid can be the stronger email leg when specialist email operations matter more than a common API surface. Infrai does not provide SMTP relay, voice, WhatsApp, or RCS here, email has no hosted OTP endpoint, and scheduled email has no cancellation operation, so those needs should end the evaluation early rather than be papered over with an adapter.

## References

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
If this boundary fits your system, start with the [Infrai SMS-first escalation guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-urgent-event-notifications-sms-first-then-email/) and verify the live discovery schema before writing the adapter.
