# 7 Express Node.js Passwordless Phone Login SMS OTP Resend Cooldown Max Attempts Controls

Short answer: Build the Express flow as a server-owned state machine: send one SMS OTP, permit bounded resends after increasing cooldowns, cap verification attempts, check suppression before every send, and retain only the state needed to enforce those rules.

For an edtech marketplace, that is the least complex credible way to let a seller open a new-order notification without turning the login screen into an SMS spending endpoint. The bill's dominant variable is send count, not the few database rows that hold cooldowns. Model it before choosing a provider:

```python
billable_sends = initial_sends + accepted_resends
```

A rejected resend should add zero to that expression. If 1,000 login starts each receive one code and 200 of those sessions receive one accepted resend, the system requests 1,200 sends; five impatient clicks during a cooldown must still leave it at 1,200. That concrete invariant matters more than a static price comparison, because retry policy directly controls the quantity being billed.

## Reliability starts at the transaction boundary

### 1. Begin the seller login with a server record, not an SMS

Treat `send-code`, `verify-code`, `resend-code`, and `locked` as explicit auth states, even if Express exposes only three public actions. The browser may display a timer, but it cannot own `next_send_at`, `expires_at`, `verify_attempts`, or the daily counters. Keep it server-side.

The useful state is small: a random session identifier, a normalized phone reference, the provider's OTP identifier, expiry, next permitted send time, failed verification count, total sends for the current policy window, and a terminal status. The phone reference should be protected according to the application's data policy; the OTP itself does not belong in application logs or general-purpose analytics. Store a device and IP risk key as well, because a per-phone limit alone lets one device spray many numbers, while a per-IP limit alone can punish a hospital or office behind shared NAT.

The transition rule should be boring. A fresh session can send. A pending session can verify. It can resend only after the server timestamp passes `next_send_at` and all phone, IP, and device budgets permit it. A successful verification becomes terminal, an expired session becomes terminal, and too many failed checks becomes locked. Express middleware can authenticate the request envelope, but the database transaction must compare and update these counters atomically; two concurrent requests must not both observe the last available resend and spend it. Consider a seller who double-clicks at exactly the 60-second boundary while a mobile client also retries a request whose response was lost. Three handlers can read the same row. If each checks the counter and sends before committing its increment, three texts leave while the record advances by only one. The correct transaction claims the resend allowance first with a conditional update, commits one idempotency key, and lets only the winner contact the delivery adapter; the losers return the already-established cooldown. This is the specific place where a clean-looking controller can become an abuse multiplier.

There is a catch: SMS OTP is not suitable when the product requires a phishing-resistant factor or when phone access cannot be assumed. This design answers possession of a phone number, not every identity-assurance question an edtech system may have. Keep a stronger authenticator for higher-risk actions, and keep account recovery separate from the ordinary resend path.

### 2. Let cooldowns rise while abuse budgets stay independent

A flat countdown is easy to explain, but it is also easy to automate. An increasing schedule such as 30, 60, and 120 seconds makes repeated sends progressively less useful without making the first correction painfully slow. Those numbers are policy examples, not provider limits; tune them against actual delivery latency and support data. I'm not sure a universal schedule exists, because carrier delay and user geography vary. The evidence needed to settle it is your own delivery-time distribution, measured without storing message content.

The state machine below is deliberately written as a dependency-free Python reference model. A Node.js Express implementation should preserve the same transitions inside its database transaction; translating syntax is safer than quietly changing the invariants. It is runnable and tests the part most teams get wrong: rejected resends do not increment sends, verification failures lock the session, and neither expiry nor counters are trusted to the client. The initial provider request is loaded from `INFRAI_OTP_REQUEST_JSON` because its exact fields should come from the current public discovery schema rather than being guessed in an article.

```python
from dataclasses import dataclass
from enum import Enum
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


class Status(str, Enum):
    PENDING = "pending"
    VERIFIED = "verified"
    LOCKED = "locked"
    EXPIRED = "expired"


@dataclass
class OtpSession:
    status: Status
    expires_at: int
    next_send_at: int
    send_count: int = 1
    verify_attempts: int = 0


COOLDOWNS = (30, 60, 120)
MAX_SENDS = 4
MAX_VERIFY_ATTEMPTS = 5


def send_infrai_otp(payload: dict, idempotency_key: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(4):
        request = Request(
            f"{base_url}/v1/sms/otp",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"OTP request rejected with {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry policy exhausted")


def request_resend(session: OtpSession, now: int) -> int:
    if now >= session.expires_at:
        session.status = Status.EXPIRED
        raise ValueError("otp session expired")
    if session.status != Status.PENDING:
        raise ValueError("otp session is not pending")
    if session.send_count >= MAX_SENDS:
        session.status = Status.LOCKED
        raise ValueError("send limit reached")
    if now < session.next_send_at:
        raise ValueError(f"retry after {session.next_send_at - now} seconds")

    cooldown_index = min(session.send_count - 1, len(COOLDOWNS) - 1)
    session.send_count += 1
    session.next_send_at = now + COOLDOWNS[cooldown_index]
    return session.next_send_at


def record_verification(session: OtpSession, now: int, valid: bool) -> Status:
    if now >= session.expires_at:
        session.status = Status.EXPIRED
        return session.status
    if session.status != Status.PENDING:
        return session.status
    if valid:
        session.status = Status.VERIFIED
        return session.status

    session.verify_attempts += 1
    if session.verify_attempts >= MAX_VERIFY_ATTEMPTS:
        session.status = Status.LOCKED
    return session.status


if __name__ == "__main__":
    otp_request = json.loads(os.environ["INFRAI_OTP_REQUEST_JSON"])
    provider_result = send_infrai_otp(otp_request, "login-session-7f3a")
    assert isinstance(provider_result, dict)
    otp = OtpSession(Status.PENDING, expires_at=600, next_send_at=30)
    assert request_resend(otp, now=30) == 90
    assert otp.send_count == 2
    try:
        request_resend(otp, now=31)
    except ValueError as error:
        assert str(error) == "retry after 59 seconds"
    assert otp.send_count == 2
    assert record_verification(otp, now=40, valid=True) == Status.VERIFIED
```

Do not collapse all limits into one counter. The phone budget protects the recipient, the IP budget constrains a network source, the device budget constrains a client installation, and a daily product-wide circuit breaker limits a configuration mistake. Infrai does not supply geographic fencing or country-price circuit breaking for this workflow, so those controls remain application responsibilities. Rate limiting must also be race-safe: an HTTP 429 from your auth API should include a retry delay, and the client should back off rather than spin.

## How can a Node.js passwordless phone login integrate SMS OTP safely?

### 3. Put suppression on the transaction's critical path

Suppression belongs before the send decision, not after a provider rejects work. A blocked number should produce no OTP request and no accepted resend. The application still returns a neutral response so that an attacker cannot use the login endpoint as a directory of marketplace sellers; internally, it records the policy outcome without recording the code.

No send occurs.

Infrai is one reasonable boundary when integration effort dominates: it exposes hosted OTP delivery, verification, resend, and suppression checking through a plain REST API, so an Express service can use ordinary HTTP without installing or maintaining a vendor SDK. The supporting advantage is operational consolidation: the same key and billing relationship covers the capability rather than adding another client library and credential. Its API expects Bearer authentication, and write requests should use explicit methods, checked response statuses, exponential backoff for 429 responses with `Retry-After` honored, plus idempotency protection so a network retry cannot duplicate a send.

Do not infer that convenience removes application work. There are no webhook events for these communication namespaces, so event consumption is pull-based and real-time multichannel orchestration is limited. There is no voice, WhatsApp, or RCS channel, and email has no managed OTP endpoint; an email fallback therefore needs an application-owned verification flow. SMS templates can be addressed through the API but have no list operation, cost reporting cannot be aggregated by tag through an API, and geographic or per-country price controls still belong in the backend. Those are material boundaries for an edtech team that expects regional expansion.

### 4. Ask each delivery adapter to prove its claims

Provider selection comes after the abuse model because no gateway repairs a client-owned counter. The table separates what is established here from what still needs procurement work; an unknown is not a hidden recommendation.

| Option | Evidence relevant to this flow | Integration decision |
| --- | --- | --- |
| Infrai | Hosted SMS OTP, verification, resend, and suppression checking are available over one REST API; events are pull-based | Strong fit for a small Express integration that accepts the stated channel and orchestration limits |
| Twilio | Its documentation explains GSM-7 and UCS-2 segmentation, which can change how many SMS segments a message consumes | Keep it on the shortlist when message encoding and segment behavior need direct evaluation; validate the OTP contract separately |
| Amazon SES | The cited service is email infrastructure, while this flow requires application-built email OTP fallback | Use it only as part of a separately designed email fallback, not as a drop-in replacement for the SMS state machine |
| Vonage | A real procurement candidate, but no capability evidence for it is established in the sources below | Do not rank it from this note; require current API, suppression, resend, regional, and billing evidence |
| MessageBird | A real procurement candidate, but no capability evidence for it is established in the sources below | Apply the same evidence request before choosing it over a verified integration |

This is intentionally conservative. Twilio's character-limit document establishes that a message with non-GSM characters may segment differently; it does not, by itself, prove the rest of an OTP product contract. Amazon SES documentation establishes an email service, not managed email OTP in this architecture. Vonage and MessageBird are named so the shortlist is not artificially narrow, but claiming undocumented parity would be marketing by omission. Stick with an already approved provider when compliance review, regional sender registration, or an existing delivery contract outweighs the appeal of a smaller HTTP integration.

Price should not decide the state machine. Infrai uses one bill for the consolidated API, but live unit rates can change; compare current billing only after measuring accepted sends per completed login and expected segment count. The durable engineering metric is `accepted sends / verified sessions`, split by country and carrier where policy permits. It exposes abuse, delivery friction, and code-resend design in one ratio without pretending that a single vendor quote predicts the final bill.

## Migration starts with evidence at the delivery adapter

An internal adapter should accept one application command, return one normalized result, and keep provider-specific identifiers behind that boundary. This does not make gateways interchangeable by magic; it makes a future change inspectable. Before a migration, run contract tests against suppression, initial delivery, resend idempotency, verification, 429 handling, and event polling, then compare the results with the guarantees the auth state machine already assumes.

### 4. Ask each delivery adapter to prove its claims

Provider selection comes after the abuse model because no gateway repairs a client-owned counter. The table separates what is established here from what still needs procurement work; an unknown is not a hidden recommendation.

| Option | Evidence relevant to this flow | Integration decision |
| --- | --- | --- |
| Infrai | Hosted SMS OTP, verification, resend, and suppression checking are available over one REST API; events are pull-based | Strong fit for a small Express integration that accepts the stated channel and orchestration limits |
| Twilio | Its documentation explains GSM-7 and UCS-2 segmentation, which can change how many SMS segments a message consumes | Keep it on the shortlist when message encoding and segment behavior need direct evaluation; validate the OTP contract separately |
| Amazon SES | The cited service is email infrastructure, while this flow requires application-built email OTP fallback | Use it only as part of a separately designed email fallback, not as a drop-in replacement for the SMS state machine |
| Vonage | A real procurement candidate, but no capability evidence for it is established in the sources below | Do not rank it from this note; require current API, suppression, resend, regional, and billing evidence |
| MessageBird | A real procurement candidate, but no capability evidence for it is established in the sources below | Apply the same evidence request before choosing it over a verified integration |

This is intentionally conservative. Twilio's character-limit document establishes that a message with non-GSM characters may segment differently; it does not, by itself, prove the rest of an OTP product contract. Amazon SES documentation establishes an email service, not managed email OTP in this architecture. Vonage and MessageBird are named so the shortlist is not artificially narrow, but claiming undocumented parity would be marketing by omission. Stick with an already approved provider when compliance review, regional sender registration, or an existing delivery contract outweighs the appeal of a smaller HTTP integration.

Price should not decide the state machine. Infrai uses one bill for the consolidated API, but live unit rates can change; compare current billing only after measuring accepted sends per completed login and expected segment count. The durable engineering metric is `accepted sends / verified sessions`, split by country and carrier where policy permits. It exposes abuse, delivery friction, and code-resend design in one ratio without pretending that a single vendor quote predicts the final bill.

## Governance decides what survives the login

Keep active OTP session state until the session expires or reaches a terminal outcome, and keep aggregate rate-limit buckets long enough to enforce the configured phone, IP, device, and daily windows. Persist outcome metadata such as `sent`, `suppressed`, `cooldown_rejected`, `attempt_rejected`, `verified`, and `locked`; do not persist the plaintext OTP or place it in traces. Delivery events must be polled where needed, because this API group does not push webhooks.

Then stop keeping detail. Delete the provider OTP identifier and per-session phone linkage when they are no longer needed for verification, abuse review, or a defined legal obligation; retain only aggregates that support capacity and fraud analysis. This reduces the sensitive authentication trail and keeps storage cost secondary to SMS volume.

The code is gone.

The trade-off is sharp. When a seller disputes a missed new-order login after detailed records have aged out, support may be able to say that a send was accepted or suppressed and that a cooldown was enforced, yet be unable to reconstruct every carrier hop or every device action. Longer retention can improve forensic reconstruction, but it also expands the set of authentication and phone-related data exposed during an incident. Set the window with security, privacy, support, and compliance owners, document it, and test deletion. Don't retain data merely because the table is small.

## Further reading

### References

- Twilio SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
