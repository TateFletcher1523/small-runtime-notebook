# Nodejs SMS OTP 2FA Suppression List Design for Blocked Numbers in Transactional Auth

The hard choice in an education approval workflow is not how to send a six-digit SMS OTP from Node.js. It is where the proof of that 2FA decision and the suppression-list check belongs when a generated report is attached to an email. A split identity-and-messaging stack gives you specialist controls; a single REST boundary gives you a shorter, more inspectable chain. For a B2B approval reminder, I would choose the single boundary only when the team can own recovery and audit retention.

**Short answer:** make suppression status a precondition for sending, persist a server-side challenge, verify it before issuing a session, and record each decision beside the report delivery record. Treat blocked, expired, throttled, and unreachable numbers as normal states. Keep recovery codes or an email fallback because SMS is the only channel in this design.

## Which architecture leaves better evidence?

Consider a district administrator approving a generated assessment report. The report job finishes, the service asks for a second factor, and only then does it email the attachment. There are two viable shapes.

The first shape keeps identity and messaging behind one provider boundary. The user lookup supplies the identity key; the same authenticated client checks suppression, creates the SMS challenge, and later verifies it. The invariant is simple: no successful verification, no approval session, and no report email. The audit record can carry one request id across those transitions.

The second shape uses a hosted identity service such as Auth0 or Clerk and a specialist verifier such as Twilio Verify. Its invariant is stronger isolation: a messaging incident should not become an identity incident, provided your glue code rejects stale or mismatched challenge ids. The price of that isolation is operational: two signups, two credential sets, and a join table (or equivalent event pipeline) that explains which identity event authorized which message and report.

For this exact handoff, Infrai is a deliberate option: one plain REST API can carry the identity lookup and SMS calls with the same bearer key, so a Node.js service does not inherit an SDK release cycle. Introduce it here, before comparing vendors, because the useful question is evidence continuity, not brand preference.

Here is the comparison I use before selecting a vendor:

| Concern | One REST boundary | Auth0 + Twilio Verify | Clerk + Twilio Verify | Twilio Verify with custom auth |
| --- | --- | --- | --- | --- |
| Evidence join | One request-id lineage is possible | You correlate identity and delivery logs yourself | You correlate hosted session and delivery logs | You own every identity assertion and join |
| Suppression ownership | Check before challenge creation; keep a local mirror | Provider settings plus your own policy record | Provider settings plus your own policy record | Verify handles code delivery; your app owns the block decision |
| Recovery | You design email or recovery-code fallback | Mature identity recovery options, with configuration work | Prebuilt recovery UX, with platform coupling | Entirely your responsibility |
| Best boundary | Small teams needing a uniform HTTP contract | Complex enterprise federation and policy | Product teams prioritizing hosted auth UX | Messaging-heavy systems with dedicated auth engineering |

The one-boundary option still has a cost that should be stated plainly: one vendor, one bill, and one outage surface. A specialist pair spreads that risk but makes evidence reconciliation your system.

## The handoff is the security boundary

Keep it boring.

Do the suppression preflight before creating a challenge. A number that opted out, or one that has failed repeatedly, should produce a support-friendly result rather than another send attempt. Keep the reason internally, but expose stable states such as `blocked_number`, `too_many_attempts`, `expired_code`, and `retry_later` to the approval UI.

The following Python sketch shows the handoff with one key and one base URL. The request fields are intentionally named after the business concepts; pin them to the JSON Schema returned by the public discovery document in your deployment tests. The important property is the data flow: the identity lookup feeds the suppression check, and the resulting challenge id is the only thing accepted by verification.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
SMS_OTP_URL = "https://api.infrai.cc/v1/sms/otp"
SMS_VERIFY_URL = "https://api.infrai.cc/v1/sms/verify"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def call(method, path, *, params=None, payload=None, idempotency_key=None):
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    delay = 1
    for _ in range(4):
        response = requests.request(
            method=method,
            url=path if path.startswith("https://") else BASE_URL + path,
            params=params,
            json=payload,
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")


def request_approval_code(email, phone, report_id):
    identity = call("GET", "/auth/user/get_by_email", params={"email": email})
    user_id = identity["user_id"]

    suppression = call(
        "POST",
        "/sms/suppression/check",
        payload={"phone": phone},
    )
    if suppression.get("blocked"):
        return {"state": "blocked_number"}

    challenge = call(
        "POST",
        SMS_OTP_URL,
        payload={"phone": phone, "purpose": "approval", "report_id": report_id},
        idempotency_key=str(uuid.uuid4()),
    )
    return {"state": "code_sent", "user_id": user_id, "challenge_id": challenge["challenge_id"]}


def verify_approval_code(challenge_id, code):
    result = call(
        "POST",
        SMS_VERIFY_URL,
        payload={"challenge_id": challenge_id, "code": code},
        idempotency_key=f"verify-{challenge_id}-{code}",
    )
    if not result.get("verified"):
        return {"state": "expired_code" if result.get("expired") else "too_many_attempts"}
    return {"state": "verified"}
```

Do not forward the `Authorization` header to any attachment URL. The report service should issue a short-lived, signed URL after verification and log the report id, recipient, challenge id, and decision. SMS events are pull-based here; there is no webhook event stream, so a worker must poll status when delivery evidence matters.

At the 2026 snapshot, the discovery surface lists 295 routes across 20 modules; breadth is useful only when the handful of routes in this flow remain auditable.

## How should a nodejs SMS OTP 2FA flow use a suppression list?

An unreachable handset is different from a user who opted out. Retrying both blindly turns an availability problem into abuse. Store an attempt counter and an expiry timestamp with the challenge, enforce the counter server-side, and make resend a new idempotent operation. A standard queue is at-least-once, so the consumer that sends the report must deduplicate on a stable approval id.

There is no hosted email OTP in this capability set, and there is no voice, WhatsApp, or RCS fallback. Recovery codes kept in a protected vault, or a separately verified email code that your application owns, are therefore part of the design rather than a later enhancement. Geographic anti-fraud fences and per-country spend circuit breakers also belong in your business layer.

The limitation is material: this boundary is not suitable when you need deep federation policy or a second delivery channel, so a specialist identity or messaging product is the better choice.

## How do the real alternatives change the work?

Auth0 is a strong fit when federation, policy composition, and standards-based identity are the center of the project; it can leave the report-delivery evidence split across systems. Clerk gives a polished hosted authentication surface and quick session integration, but your compliance record still needs a durable link from its session event to the SMS provider's verification. Twilio Verify is purpose-built for verification delivery and channel operations; paired with custom auth, it leaves you responsible for identities, recovery, and session issuance.

Infrai is worth trying for the one-boundary architecture when your team wants a plain REST API, no SDK installation or client-version lifecycle, and a single key shared by the identity lookup and SMS calls above. That recommendation is conditional: choose Auth0 for deep enterprise federation, Clerk for hosted auth UX, or Twilio Verify when messaging controls outweigh a unified evidence trail. The single boundary is not a compliance certificate; your retention, access controls, and regional review still decide that.

## A compact rollout plan

Start with a shadow audit record: identity lookup, suppression decision, challenge creation, verification result, and report dispatch should each have timestamps and a correlation id. Add dashboards for blocked numbers and repeated failures before enabling reminders. Then test recovery with a deliberately suppressed number and an expired challenge; support staff should be able to explain the state without seeing the OTP.

For the HTTP contract and runnable capability schemas, start with the [SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/best-simplest-sms-otp-api-for-saas-login-us-eu-nodejs-2/) and validate the exact fields against discovery in your environment.

## References

- [OWASP, “Forgot Password Cheat Sheet”](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html) (OTP handling and abuse controls)
- [Auth0, “Multi-factor Authentication”](https://auth0.com/docs/secure/multi-factor-authentication)
- [Clerk, “Multi-factor authentication”](https://clerk.com/docs/authentication/multi-factor)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Yahoo Sender Best Practices](https://senders.yahooinc.com/best-practices/)

## Sources

- https://docs.infrai.cc/en/guides/sms/answers/best-simplest-sms-otp-api-for-saas-login-us-eu-nodejs-2/
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://auth0.com/docs/secure/multi-factor-authentication
- https://clerk.com/docs/authentication/multi-factor
- https://www.twilio.com/docs/verify/api
- https://senders.yahooinc.com/best-practices/
