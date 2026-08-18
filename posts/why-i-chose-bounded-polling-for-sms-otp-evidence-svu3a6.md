# Why I Chose Bounded Polling for SMS OTP: Evidence Before Real-Time UX

For a logistics portal that emails a generated report as an attachment, I choose bounded status polling for SMS OTP when webhooks are unavailable, then stop and offer a clear retry path. The decision is about compliance evidence first and “real time” second: a login screen must prove what it knows, avoid turning an SMS delay into an account-enumeration oracle, and leave an audit trail that survives a support ticket.

Short answer: poll a provider-neutral status endpoint on a short, capped schedule, treat “sent” as different from “delivered,” and never make delivery a prerequisite for a safe timeout. Use a push or webhook path when the provider and network can meet your evidence requirements; polling is a reasonable fallback, not a promise of instant delivery.

## The invariants I put in the decision record

The OTP is a short-lived secret, not a delivery receipt. I generate it server-side, store only a verifier and an expiry, bind it to the login attempt, and accept it once. OWASP’s reset guidance calls out single-use, rate limits, and consistent responses; those are the same controls I want for 2FA, even though the user is signing in rather than resetting a password.

No magic timer.

For the report workflow, the audit event records the attempt ID, channel, creation time, expiry, status transitions, and the eventual verification result. It does not record the OTP itself or put a phone number in a client-visible status URL. A support engineer can answer “what happened?” without receiving another secret.

There are hard boundaries here. A provider saying “accepted” means it accepted a request; it does not mean the handset displayed anything. A carrier may delay, filter, or reorder messages. “Real time” is therefore a UI behavior with a timeout, not a property I can infer from one API response.

## How should polling shape SMS OTP, 2FA, and real-time UX?

The client starts one attempt and polls an opaque, authenticated attempt resource. I use backoff, jitter, and a deadline, for example 1, 2, 4, 8, and 10 seconds, capped at 30 seconds overall for an interactive login. The exact numbers belong in an experiment and a runbook; your mileage may vary by carrier mix in the US and EU. I’m not sure a longer window improves completion enough to justify holding the user on one screen, so the UI offers “try another method” after the deadline.

The response is deliberately boring: `pending`, `accepted`, `delivered`, `expired`, or `failed`. Only `delivered` is a delivery signal, and even that is not proof of human reading. Verification remains the authority. The browser never polls by phone number, and it cannot ask whether an arbitrary account has a live OTP.

That distinction matters.

Here is the critical path in Python. The transport is intentionally generic; the important parts are bounded work, an idempotent attempt ID, and separate verification.

```python
import random
import time


def wait_for_delivery(client, attempt_id, deadline=30.0):
    started = time.monotonic()
    delay = 1.0
    while time.monotonic() - started < deadline:
        state = client.get_status(attempt_id)  # authenticated, opaque ID
        if state in {"delivered", "failed", "expired"}:
            return state
        time.sleep(delay + random.uniform(0, 0.25))
        delay = min(delay * 2, 8.0)
    return "timeout"


def verify_code(store, attempt_id, submitted_code):
    record = store.load(attempt_id)
    if record is None or record.is_expired or record.used:
        return False
    if not store.matches_verifier(record, submitted_code):
        store.note_failed_verification(attempt_id)
        return False
    store.mark_used(attempt_id)
    return True
```

That timeout is a UX boundary, not a cleanup shortcut. A background job can reconcile late delivery events, while the login attempt remains expired and unusable. This prevents a delayed SMS from becoming a valid credential after the user has moved on.

## What fails when there is no webhook?

Polling shifts work to the client and the status service. Mobile radios sleep, tabs are throttled, and a corporate proxy can make a five-second interval optimistic. Aggressive polling also creates a small denial-of-service multiplier during a carrier incident. I set per-attempt and per-account limits, return `429` with a retry hint, and emit metrics for poll volume, status latency, verification success, and abandonment.

The failure pattern I look for in a staging trace is easy to miss: the first request is accepted at 09:00:01, the browser polls four times, a resend is issued at 09:00:12, and the original text arrives at 09:00:28. If both verifiers remain live, the user can enter the wrong message while the support log appears to show a healthy delivery. If the client treats every non-`delivered` response as a hard failure, the same person may create a third attempt, multiplying cost and rate-limit pressure. I therefore make the attempt state machine authoritative on the server, invalidate the old verifier at resend, attach every transition to one correlation ID, and test delayed, duplicated, and out-of-order status responses. The UI can stay calm because it is reporting the state of one attempt rather than guessing what a handset did.

The opposite failure is quieter: a “pending” state that the UI interprets as failure makes people request three codes. Only the newest code should be valid, and every resend should invalidate the previous verifier. If the provider has no delivery status at all, I stop pretending to offer one; the UI says the message was requested and gives a bounded wait plus another factor.

For evidence, I retain immutable event records with a correlation ID and clock source, redact message bodies, and restrict operator access. In the EU, that supports data-minimization and purpose-limitation work under GDPR; in the US, it helps demonstrate an accountable authentication process, while messaging campaigns may also trigger rules such as the FTC’s CAN-SPAM requirements. The exact legal basis depends on the service and jurisdiction, so counsel should validate retention and notice language.

## Trade-offs I would record before shipping

| Approach | Evidence quality | UX behavior | Failure boundary | Choose it when |
| --- | --- | --- | --- | --- |
| Bounded polling | Captures observed status and timestamps | Responsive for a short window | Client/network load; no event push | Webhooks are unavailable or blocked |
| Webhook plus fallback poll | Strong event trail with reconciliation | Fast when events arrive | Signature, replay, and delivery handling | The provider can sign events and the team can operate them |
| Fixed delay, then verify | Minimal moving parts | Simple but opaque | Delays do not equal delivery | A low-volume internal tool tolerates ambiguity |
| Voice or authenticator fallback | Independent channel evidence | Can recover from SMS filtering | More enrollment and support work | Risk or carrier performance makes SMS unsuitable |

The catch is that polling is not suitable when regulators or an incident process requires a provider-signed, near-real-time event. Stick with a webhook-capable design in that case, and keep a reconciliation poll for missed events. Conversely, a small internal fleet dashboard may reasonably skip delivery polling and let verification plus a visible timeout carry the interaction.

I also avoid making “US versus EU” a country-code branch in the browser. Keep policy server-side: allowed channels, retention, consent or notice text, and regional data residency should be configuration with an audit history. Phone-number normalization, quiet hours, and carrier filtering belong in the messaging layer, where they can be tested against real regional data.

## A release checklist for the report portal

Before rollout, I run carrier-delay and duplicate-message tests, freeze a clock for expiry tests, and replay webhook or poll responses to prove idempotency. I verify that logs contain no OTP, that a `404` does not reveal whether a user exists, and that resend limits hold across devices. The dashboard has one useful question: how many attempts reached verification before expiry, by region and channel?

Three words matter: requested is not delivered. That distinction keeps the email attachment workflow auditable without making a carrier’s timing someone’s password policy.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://gdpr-info.eu/art-5-gdpr/
- https://www.rfc-editor.org/rfc/rfc6238
