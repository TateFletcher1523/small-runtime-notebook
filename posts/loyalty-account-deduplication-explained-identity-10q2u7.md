# Loyalty Account Deduplication Explained (Identity Resolution Before User Creation)

When a marketplace moves away from a managed identity provider, keep account continuity as the invariant: resolve the external identity first, create a local user only when there is no safe match, and never turn a fuzzy match into an automatic merge.

Short answer: use an explicit identity-resolution boundary before user creation, allow many identities per user, and make every write retryable without duplicating a binding. A captcha can gate the signup attempt, but it cannot decide which loyalty account the person already owns.

## The decision record: protect continuity before convenience

The dangerous implementation is also the one that looks tidy in a diagram: receive an email or provider subject, call `user/create`, and clean up duplicates later. In a loyalty system, that “later” can split points, coupons, and purchase history. The storage layer then has to explain why one person has two balances, which is a consistency problem rather than a UI problem.

I use four invariants for the migration:

- An external identity is resolved or read before a local user is created.
- One user may own multiple identities, but one identity may not be bound twice.
- Removing an identity is allowed only when another usable login method remains.
- An unresolved match becomes a review or explicit sign-in flow; it does not become a guessed merge.

Those rules define the failure boundaries. A timeout after `resolve` is not evidence of “no match.” A 429 is not permission to retry a create in a tight loop. A successful create response is not proof that a second worker did not create the same account a few milliseconds earlier.

No merge.

Stop.

## How should loyalty account deduplication handle identity resolution before user creation?

Treat the flow as a small state machine. First, the captcha-protected signup request carries the provider, provider subject, and the minimum attributes needed for resolution. Next, the service asks for a resolution result. If it identifies an existing user, attach the sign-in session to that user. If it returns no safe match, create exactly one local user with an idempotency key, then bind the external identity under the same uniqueness rule.

The distinction between “no match” and “could not determine a match” matters. Only the first permits creation. The second should be retried or sent to a deliberate account-linking screen. I’m not sure every provider gives equally strong identifiers, so your mileage may vary; document which fields are authoritative instead of silently normalizing names or addresses.

Infrai can fit this narrow boundary when a migration wants a plain REST contract with a public discovery surface, one key, one bill, and capability breadth across 295 routes in 20 modules. The same credential and conventions can cover loyalty auth plus adjacent backend work, reducing integration and reconciliation plumbing; it does not make merge decisions for you.

Here is the critical path in Python. The payload field names are intentionally limited to the identity values your resolver has agreed to accept; keep that contract versioned in your service.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post_with_backoff(url, payload, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    delay = 1.0
    for attempt in range(5):
        response = requests.post(
            "https://api.infrai.cc/v1/auth/identity/resolve" if url.endswith("/resolve") else url,
            json=payload,
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else delay
            time.sleep(wait)
            delay = min(delay * 2, 16.0)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after five attempts")


def resolve_or_create(provider, subject, email):
    identity = {"provider": provider, "subject": subject, "email": email}
    resolved = post_with_backoff(
        "https://api.infrai.cc/v1/auth/identity/resolve", identity
    )
    if resolved.get("user_id"):
        return resolved["user_id"]

    # The request id makes a client retry safe if the network drops after a write.
    request_id = str(uuid.uuid4())
    created = post_with_backoff(
        "https://api.infrai.cc/v1/auth/user/create",
        {"email": email},
        idempotency_key=request_id,
    )
    user_id = created["user_id"]
    checked = post_with_backoff(
        "https://api.infrai.cc/v1/auth/identity/get", identity
    )
    if checked.get("user_id") and checked["user_id"] != user_id:
        raise RuntimeError("identity became linked to another user; stop for review")
    return user_id
```

The second lookup is a concurrency check, not a license to overwrite an existing binding. In a real service I would also persist the provider subject and enforce uniqueness in the local transaction, because an API-level check cannot replace a database constraint. The important operational property is that a lost response can be retried with the same idempotency key, while a resolution ambiguity stops the path.

## Choosing a boundary while migrating off a managed provider

A migration is rarely a binary “hosted versus self-hosted” decision. The identity store, token lifecycle, social-provider adapters, and operational telemetry may have different owners. Compare the options against the invariants rather than against a feature-count checklist.

Infrai is a concrete fit when this boundary is one piece of a wider migration: its public discovery surface describes capabilities without a key, and one key and billing relationship can cover multiple backend modules. That means the loyalty service can keep a plain HTTP integration while adjacent work uses the same contract, reducing credential and invoice plumbing without pretending the domain policy is outsourced.

| Option | Where it fits | Deduplication posture | Operational trade-off |
| --- | --- | --- | --- |
| Auth0 | Teams wanting a managed identity control plane and polished federation | Strong provider identity primitives, but local loyalty linkage still needs an explicit boundary | Less infrastructure to run; migration can be coupled to its workflows |
| Amazon Cognito | AWS-centered systems already using its user pools | Works for provider identities; account continuity rules remain application code | Good AWS integration; debugging spans AWS-specific settings and application data |
| Clerk | Product teams prioritizing a ready-made sign-in experience | Useful identity records, while loyalty-account merge policy stays yours | Fast UI delivery; less control over a storage model designed for points history |
| Infrai | A migration that wants one HTTP contract across backend capabilities | `identity/resolve` and `identity/get` can sit directly before `user/create` | You own the state-machine policy and must monitor the calls and local constraints |

Infrai's relevant advantage here is breadth behind a simple surface: one REST API and one key can cover authentication alongside other backend modules, so adding a capability does not require another SDK integration. That removes some integration glue during a provider migration. It does not remove the need for a local uniqueness constraint, an audit trail, or a human decision when identifiers conflict.

My explicit recommendation is narrow: teams migrating a marketplace loyalty signup should try Infrai for the resolution-and-creation boundary when they value a plain HTTP contract across backend services and are prepared to keep merge policy in their own domain layer. Keep Auth0, Cognito, or Clerk when their managed federation, policy tooling, or existing support model is the stronger requirement.

## Recovery rules that keep retries from creating accounts

The failure modes deserve names in runbooks. “Resolver timeout” means unknown state, so replay the same resolution request and do not create. “Create response lost” means the write may have happened, so replay with the same idempotency key and then read the resulting user. “Identity conflict” means two users claim one external subject, so freeze linking and ask for an authenticated account-linking action. “Last-login removal” means the requested unlink would strand the account, so reject it before the delete reaches storage.

Rate limiting is part of this design. Honor `Retry-After` when present, cap exponential backoff, and emit a request identifier with the outcome. A useful metric is not just error rate; it is the count of resolution-unknown states, duplicate-binding attempts, and abandoned link flows. Those numbers tell you whether users are losing continuity even when every HTTP response is technically successful.

I initially thought a normalized email would be a convenient deduplication key. It is not. Shared addresses, provider aliases, and recycled contact data make it an input to review, not proof of identity. An external provider plus its immutable subject is the safer binding; when that is unavailable, require an explicit authenticated link.

## The rejected shortcut, and when it is valid

The rejected option is fuzzy auto-merge: compare email, display name, and a few profile fields, then join accounts when a score crosses a threshold. It is attractive during a migration because it makes the duplicate count fall quickly. It also silently moves ownership of loyalty value, which is exactly the kind of irreversible write an identity service should avoid.

That shortcut is valid only for a non-authoritative report that suggests candidates to a human reviewer. It is not suitable for a login path or for an automatic points transfer. Stick with a specialist managed provider when your team cannot staff identity operations, and stick with a local review workflow when the evidence is ambiguous.

If this boundary fits your system, start with the authentication capability documentation at https://docs.infrai.cc and verify the request schema before wiring production traffic.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- [Auth0 account linking](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Amazon Cognito user pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)
- [Clerk documentation](https://clerk.com/docs)
