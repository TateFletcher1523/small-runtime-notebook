# Node.js Edtech Contact Routing with Onboarding Welcome Email API Deliverability

Short answer: choose an email API only after proving that it can sit behind a durable message state machine. For an edtech contact form, the decisive integration test is whether one submission can be assigned to exactly one support queue, produce at most one welcome message, and absorb a later bounce without letting a retry contact a suppressed address. DKIM and SPF matter, but neither repairs weak ownership of state.

This architecture decision records a narrow choice: keep routing, idempotency, and suppression in the application data layer; treat the delivery API as an asynchronous transport with replaceable credentials and authenticated callbacks. The provider may accept a message before the receiving domain accepts it, so `accepted` is not `delivered`. Persist that distinction.

## How should an email API handle onboarding welcome message deliverability?

A beginner integration often sends mail directly inside the Express request that accepts the contact form. The happy path is short. The failure boundary is not: the process can lose its response after the remote API accepted the message, a user can submit twice, and a bounce can arrive after an agent has already replied.

Timeouts lie.

The durable record needs four invariants. A stable submission ID selects one queue. A logical welcome has one idempotency key. Suppression is checked immediately before every delivery attempt, not merely when the form is received. Provider events advance local state only when their event ID has not been consumed before. Those rules do not promise that the internet delivers exactly once; they constrain the effects the application owns.

Keep the states explicit: `queued`, `submitted`, `delivered`, `deferred`, `bounced`, and `suppressed`. A timeout leaves the record uncertain until reconciliation or a callback resolves it. Do not relabel uncertainty as failure and send again blindly.

SPF, standardized by RFC 7208, authorizes hosts to use a domain in the SMTP `MAIL FROM` or `HELO` identity. DKIM, standardized by RFC 6376, signs selected headers and the body for verification against a DNS key. They answer different questions, and neither is evidence that a mailbox displayed the message. Delivery status notifications have their own machine-readable fields under RFC 3464; preserve the reported action, status, and diagnostic rather than flattening every negative result into `bounce`. There is also a concrete SPF ceiling that beginners can miss: RFC 7208 Section 4.6.4 limits terms that cause DNS queries to 10 during one check, and exceeding that limit produces `permerror`. Deep chains of `include` mechanisms consume that budget. The number is 10, not a vague warning to keep DNS tidy.

## Decision boundaries and options

Integration effort should be measured across the whole lifecycle, including DNS verification, callback authentication, replay handling, reconciliation, and operational diagnosis. A five-line send example measures almost none of that work.

This is the trade-off.

| Boundary | Application-owned state machine | Provider-owned workflow | Synchronous send in Express |
|---|---|---|---|
| Queue assignment | One local transaction records the form and queue | Mapping may live outside the primary record | Request code selects a queue transiently |
| Duplicate submission | Stable local idempotency key | Depends on remote workflow semantics | Commonly retried without durable context |
| Suppression | Checked before each attempt; reason is retained | Convenient if every sender uses one account | Easy to omit on a new call path |
| Late delivery event | Deduplicated and applied as a state transition | Handled remotely, with export requirements | Often has no durable target |
| Provider replacement | Adapter and event mapper change | Workflow migration is part of replacement | Call sites and error handling change |
| Main cost | Schema, worker, and reconciliation logic | Dependency on remote state and controls | Hidden recovery and audit work |

The first option requires more initial code, but its failure surface is inspectable in the same database that owns the student or instructor request. Its limitation is equally real: a team must operate a worker, define transitions, retain events, and reconcile uncertain submissions. It is not suitable for a disposable experiment whose only goal is checking whether people understand the form. The provider-owned alternative reduces application code but makes remote workflow semantics, export, and access controls part of the dependency. The synchronous alternative has the lowest setup effort and the weakest recovery boundary. For this system it is rejected because an HTTP response deadline is unrelated to an SMTP delivery lifecycle.

One distinction prevents a policy mistake. A welcome message that contains only requested account or support information is not automatically the same stream as promotional mail. When a message is a qualifying mailing-list message, RFC 8058 defines one-click unsubscribe through `List-Unsubscribe` and `List-Unsubscribe-Post` headers and requires DKIM coverage for the relevant headers. Classify the stream before applying that mechanism; do not attach marketing content merely because the transport makes it convenient.

Classification comes first.

## Critical path in code

The following Python is deliberately an interface sketch even though the edge service may be Node.js and Express. The contract is the point: one database transaction creates the routed case and an outbox item, while a worker performs the external side effect later. Concrete SQL, transaction isolation, and callback signature verification remain implementation-specific and must be tested against the selected database and transport.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class WelcomeCommand:
    submission_id: str
    recipient: str
    queue: str
    template: str


class Store(Protocol):
    def suppressed(self, recipient: str) -> bool: ...
    def claim(self, key: str) -> bool: ...
    def mark_submitted(self, key: str, transport_id: str) -> None: ...
    def mark_suppressed(self, key: str, reason: str) -> None: ...


class MailTransport(Protocol):
    def send(self, *, recipient: str, template: str, key: str) -> str: ...


def deliver(command: WelcomeCommand, store: Store, mail: MailTransport) -> None:
    key = f"welcome:{command.submission_id}"

    if store.suppressed(command.recipient):
        store.mark_suppressed(key, reason="recipient_on_suppression_list")
        return

    if not store.claim(key):
        return

    transport_id = mail.send(
        recipient=command.recipient,
        template=command.template,
        key=key,
    )
    store.mark_submitted(key, transport_id)
```

`claim` must have atomic semantics in the real store. The sketch also exposes a hard interval: the process can stop after `send` succeeds but before `mark_submitted` commits. Resolve that interval with a transport-supported idempotency mechanism when its semantics are documented, or with reconciliation keyed by a stable message identifier; never assume a Python function call and a remote submission form one transaction.

Callback processing needs the same discipline. Authenticate the callback according to the selected API's documented scheme, retain the original event for audit, reject timestamps outside the chosen replay window, deduplicate by event ID, and make state transitions monotonic. A delayed `delivered` event must not erase a later, terminal suppression decision. Define precedence before production traffic supplies the awkward ordering.

Order will be awkward.

## Failure tests before selection

Do not start the evaluation with inbox screenshots. Begin with controlled faults at each ownership boundary. Send the same outbox item twice and confirm that one logical welcome is recorded. Terminate the worker after remote acceptance but before its local commit. Replay an authenticated callback. Deliver `deferred`, `delivered`, and `bounced` events out of order. Add the recipient to suppression between enqueue and send. Rotate a DKIM selector while the previous selector remains required for messages already in flight.

Then inspect evidence, not a green dashboard: database rows, immutable event payloads, DNS records, message headers, and queue depth. Set alerts on age of the oldest unsent outbox item and on callback processing failures; raw send volume can remain normal while one support queue quietly stops advancing. Retention periods and alert thresholds are local policy decisions, so choose and document them rather than copying unexplained numbers.

For the contact form itself, normalize routing inputs into a small controlled taxonomy such as `account_access`, `billing`, and `course_content`; store the submitted free text separately. Routing based directly on mutable display labels creates a fragile join between UI copy and operational ownership. Unknown categories should land in a reviewed default queue, not disappear and not guess at a student's intent.

Short messages introduce a separate compliance boundary. If the same workflow can send SMS, do not treat an email opt-out as a complete SMS consent model or vice versa. CTIA publishes messaging principles and best practices for application-to-person messaging; the architecture should preserve channel-specific consent and suppression evidence, even when one contact record contains both an email address and a telephone number.

## Rejected design and its valid use

The rejected option is a direct API call from the Express route followed by `200 OK`. It couples user-facing latency to a remote service, makes ambiguous timeouts hard to reconcile, and invites duplicate sends when clients retry. For an edtech support form, those properties conflict with the invariant that a submission has one durable queue assignment and one traceable welcome lifecycle.

Direct synchronous submission still has a valid use: a disposable prototype with no retained user data, no automated retries, and no claim of delivery guarantees can use it to validate copy and form flow. The condition is strict. Once the message participates in account onboarding, support triage, consent, or audit, the outbox and event model stop being optional plumbing.

The selection result is a contract, not a ranking: require authenticated event delivery, stable identifiers, documented bounce detail, suppression controls, DKIM support, and enough retrieval or export capability to reconcile uncertain submissions. Evaluate those capabilities with the failure tests above. The API that produces the shortest initial Express handler may still impose the largest integration effort after the first timeout.

## References

- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc3464
- https://datatracker.ietf.org/doc/html/rfc8058
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
