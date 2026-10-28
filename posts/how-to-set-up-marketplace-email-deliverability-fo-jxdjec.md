# How to Set Up Marketplace Email Deliverability for Product Events with DKIM Domain Verification

For a Nodejs product event email deliverability setup, route marketplace contact notifications through a region-aware outbox, verify the DKIM domain, and keep template ownership in the application repository while a separate adapter handles suppression, bounce polling, and the delivery API. The deciding constraint is auditability: a support team must be able to prove which approved template, domain, region, and suppression decision produced each message without treating a provider's mutable template store as the system of record.

TL;DR: sign mail with an aligned marketplace domain, verify DNS before enabling traffic, suppress known-bad recipients before submission, and reconcile asynchronous delivery events into an append-only state history. Polling is acceptable when webhooks are unavailable, provided the poller uses a durable cursor, overlaps its time window, and applies events idempotently. US and EU traffic should have explicit routing and retention rules; geography is a policy boundary, not a string appended to an endpoint.

## Decision record and invariants

The concrete workflow is a marketplace contact form: a buyer selects an order, writes a message, and expects it to reach the correct seller-support queue. The notification body may contain an order reference and a reply token, so ownership cannot be vague. Product engineering owns the versioned template and its schema. Support operations owns queue-routing data. The delivery adapter owns transport translation, never business wording. This split also determines incident ownership: application engineers can reproduce rendering from a commit and template version, support operators can explain why the order mapped to a queue, and transport operators can trace submission and delivery events without gaining the ability to rewrite approved content. If one record cannot connect those three facts by stable identifiers, the system has an audit gap even when every message happens to arrive.

Four invariants define the design. First, every accepted form submission receives a stable event ID before any mail call. Second, rendered content is derived from a reviewed template version and typed data, not edited in a remote dashboard. Third, the sender domain passes the organization's verification gate before production routing is enabled. Fourth, a permanent suppression decision wins over retries, regional failover, and manual replay.

DKIM supplies a domain-level signature that a verifier can validate through DNS, but a valid DKIM signature is not a promise of inbox placement. RFC 6376 also distinguishes the signing domain from the visible author address; operationally, that means the deployment gate should check the intended domain identity and selector rather than merely finding any valid signature. Pair DKIM with SPF and DMARC alignment, then monitor authentication results in received-message headers and aggregate DMARC reports. Do not infer durability from an API acceptance response.

The failure boundaries are deliberately narrow. DNS verification can fail without losing a contact event because the outbox remains pending. Rendering can fail without submitting malformed mail. Transport acceptance can succeed while downstream delivery later bounces. A poller can repeat a page without repeating a state transition. These boundaries matter more than provider feature lists, but they carry a real trade-off: an outbox, event ledger, and regional cursors add storage and operational work that a low-volume, non-critical notification stream may not justify.

| Template ownership model | Audit and rollback properties | Failure mode | Appropriate use |
| --- | --- | --- | --- |
| Application-owned, versioned source | Review, schema, and release travel together | Deployments are required for copy changes | Transactional contact and account messages |
| Operations-owned remote templates | Fast non-code edits | Copy can drift from the event schema or regional approval | Low-risk campaigns with a formal approval workflow |
| Fully rendered payload stored per event | Exact reconstruction of what was submitted | Retains sensitive content and increases storage obligations | Regulated evidence when retention is explicitly authorized |

The first model is the default here.

Store the template ID, immutable version, locale, content hash, and non-sensitive routing facts with the outbox record; retain the rendered body only when a documented retention requirement justifies it. This is a storage decision as much as an email decision: deletion, encryption, access control, and regional residency all become harder once free-form contact text is copied into logs.

## How should a Nodejs product event email deliverability setup handle bounces?

An accepted submission proves only that the transport took responsibility for further processing. A mailbox can reject it later, a domain can return a transient error, or a complaint can make future contact inappropriate. Model those observations as events rather than overwriting a single `status` field whose meaning changes under concurrent updates.

Use states such as `queued`, `submitted`, `delivered`, `soft_bounced`, `hard_bounced`, and `complained`, but preserve the raw transport event category and timestamp beside the normalized state. The normalization table is code-reviewed policy. It should state which observations create a permanent suppression and which schedule a bounded retry; repeated transient failures need an age and attempt ceiling, because unlimited retries turn an unavailable mailbox into a permanent workload.

Be conservative.

Hard bounces and complaints should prevent later automated submissions to that address for the same messaging purpose. A soft bounce may be retried under an explicit policy, yet a new contact-form event must not quietly bypass an existing permanent suppression. Suppression checks and enqueueing belong in one transaction or in an equivalent serialized decision, otherwise two workers can both observe “not suppressed” and submit.

## Implement the critical path

The following Python example keeps the transport generic. It assumes `outbox.reserve` atomically claims work, `suppression.lookup` reads the authoritative regional suppression set, and `transport.submit` returns a stable remote message identifier. Those interfaces are intentionally small; swapping a transport must not move template ownership or suppression policy across the boundary.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Mapping, Protocol


@dataclass(frozen=True)
class ContactEvent:
    event_id: str
    region: str
    recipient: str
    queue_name: str
    order_reference: str
    message: str


class DeliveryPort(Protocol):
    def submit(self, *, region: str, idempotency_key: str,
               sender: str, recipient: str, subject: str,
               body: str, metadata: Mapping[str, str]) -> str: ...


def deliver_contact(event: ContactEvent, templates, suppression,
                    outbox, transport: DeliveryPort) -> str:
    if event.region not in {"us", "eu"}:
        raise ValueError("unsupported processing region")

    record = outbox.reserve(event.event_id, region=event.region)
    if record.remote_message_id:
        return record.remote_message_id

    reason = suppression.lookup(event.region, event.recipient)
    if reason is not None:
        outbox.mark_suppressed(event.event_id, reason=reason)
        return "suppressed"

    template = templates.load("marketplace-contact", version="7")
    body = template.render(
        queue_name=event.queue_name,
        order_reference=event.order_reference,
        message=event.message,
    )
    content_hash = sha256(body.encode("utf-8")).hexdigest()
    remote_id = transport.submit(
        region=event.region,
        idempotency_key=event.event_id,
        sender="marketplace@example.com",
        recipient=event.recipient,
        subject=f"Contact request for {event.order_reference}",
        body=body,
        metadata={
            "event_id": event.event_id,
            "template_version": "7",
            "content_hash": content_hash,
        },
    )
    outbox.mark_submitted(event.event_id, remote_id, content_hash)
    return remote_id
```

Replace `example.com` only after the real sending domain and selector have been published and verified. During deployment, send a probe to controlled mailboxes in each enabled region, inspect authentication results, and stop promotion if alignment is wrong. DNS changes need a staged rollout because resolver caches do not share one clock; preserve an old selector while mail signed with it may still be in flight, and rotate keys by adding the new record before switching signers.

Do not trust the browser.

The contact form's routing key should be constrained data, not an address supplied by the browser. Resolve `queue_name` server-side from marketplace policy. Otherwise the mail layer becomes an open relay into arbitrary support destinations.

## Reconcile bounces with an idempotent poller

Polling requires two positions: a high-water timestamp and a stable event tie-breaker. A timestamp alone can skip events that share the same resolution or arrive late. Query with overlap, sort by `(occurred_at, event_id)`, insert each remote event under a uniqueness constraint, and advance the durable cursor only after the page commits.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class Cursor:
    occurred_at: datetime
    event_id: str


PERMANENT = {"hard_bounce", "complaint"}


def reconcile(region, event_source, ledger, suppression, cursors):
    cursor = cursors.load(region)
    since = cursor.occurred_at - timedelta(minutes=5)

    while True:
        page = event_source.list_events(
            region=region,
            since=since,
            page_token=None if cursor.event_id == "" else cursor.event_id,
            limit=200,
        )
        if not page.events:
            return

        for event in sorted(page.events, key=lambda item: (
                item.occurred_at, item.event_id)):
            with ledger.transaction():
                inserted = ledger.insert_once(
                    source=event.source,
                    event_id=event.event_id,
                    message_id=event.message_id,
                    category=event.category,
                    occurred_at=event.occurred_at,
                    region=region,
                )
                if inserted and event.category in PERMANENT:
                    suppression.add(
                        region=region,
                        recipient=event.recipient,
                        reason=event.category,
                        observed_at=event.occurred_at,
                    )
                cursor = Cursor(event.occurred_at, event.event_id)
                cursors.save(region, cursor)

        if page.next_token is None:
            return
        cursor = Cursor(cursor.occurred_at, page.next_token)
```

The five-minute overlap and page size of 200 are examples, not universal limits. That is a limitation of the sample, not a recommended service-level target. Set both from measured event lateness, API quotas, and recovery objectives, then test the boundary with multiple events at the same timestamp, duplicate pages, an empty page, a crash before cursor commit, and a crash after ledger insertion. The key assertion is uncomplicated: processing the same page twice produces one ledger event and one suppression entry. Polling also imposes detection latency and quota pressure; a signed event push can be preferable when the transport supports replay, signature verification, and durable delivery, while scheduled polling remains useful as a reconciliation backstop.

Observability should expose queue age, oldest unreconciled event age, authentication failures, bounce categories, suppression hits, polling lag, and unknown remote categories by region and template version. Avoid recipient addresses in metric labels. Alert on age and correctness indicators rather than raw traffic volume, and retain enough immutable identifiers to trace a submission without placing message bodies in routine logs.

## Deploy US and EU routes as policy boundaries

Region selection must happen before rendering and submission, using an authoritative account or marketplace policy rather than IP geolocation. Give each region its own outbox partition, credentials, event cursor, suppression namespace, and retention configuration. This does not claim that a regional endpoint alone satisfies any legal requirement; counsel and data-governance owners must define which data may cross a boundary, how long it remains, and who can access it.

Roll out one template version through a controlled sequence: validate its schema, verify signing records, submit probes, enable a small deterministic cohort, examine bounce and authentication evidence, and then expand. Rollback selects the previous template version for new events; it never mutates the historical version recorded for already submitted mail.

One subtle trap is regional failover. Sending an EU event through the US route during an outage may violate the declared processing policy and may lose access to the EU suppression set. Fail closed unless cross-region processing has been explicitly approved and suppression data is available with defined consistency. Availability is not permission.

## Rejected option and its valid use case

The rejected design stores editable transactional templates only in the delivery service and lets operations publish changes directly. It shortens the copy-edit loop, but it separates prose from the event schema, makes rollback evidence dependent on a third-party change history, and can create different unreviewed content across US and EU accounts. For marketplace contact messages, those costs outweigh the speed.

Remote template ownership is still reasonable for low-risk bulk communications when an approval workflow, immutable version history, schema validation, regional controls, and exportable audit records already exist. Even then, the application should record the selected remote version and policy decision. The boundary is risk and ownership, not ideology.

For the contact-form path, the durable choice is application-owned templates, region-scoped delivery state, and an append-only event ledger. That arrangement does not guarantee inbox placement. It does make verification, suppression, replay, and disputed-message analysis testable without granting a transport adapter authority over product language or routing policy.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 3463, Enhanced Mail System Status Codes: https://datatracker.ietf.org/doc/html/rfc3463
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
