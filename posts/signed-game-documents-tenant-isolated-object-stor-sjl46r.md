# Signed Game Documents: Tenant-Isolated Object Storage Backups and Restore Scripts

For a gaming service that retains signed documents until an explicit deletion deadline, I would start with a private cloud object-storage bucket and a small, provider-neutral restore script; self-hosted MinIO becomes reasonable only when the team already owns redundant hardware, offsite copies, and the on-call work around them. Tenant isolation is the deciding constraint, not the lowest storage quote.

Short answer: put each tenant's backup objects under an enforced namespace, keep application credentials server-side, issue short-lived signed URLs only for a named object, and test deletion and restoration as separate workflows. A beginner setup is cheap only when its operators can actually restore one tenant without exposing another tenant's documents.

## Tenant retention governance is the first boundary

The system has four invariants. A completed backup must be private, attributable to one tenant, independently restorable, and removable when its retention deadline arrives. A signed URL must grant access to exactly the intended object for a bounded interval; it must not become a second, permanent credential. Deletion must be observable, and a successful database transaction must not be mistaken for proof that every object copy is gone.

Test the boundary first.

Suppose tenant A's document is signed at 23:58, its retention deadline is 30 days later, and a support engineer needs to restore it during an incident. The application should resolve the document record, check that the caller belongs to tenant A, confirm that the deadline has not passed, and select a backup ID from an auditable list; only then should it construct an object key or request a signed URL. If the engineer changes the tenant ID in the request, the authorization decision must fail before storage is contacted. If the deadline passes while a link is being requested, the server should refuse a new link even if the object still exists. If the object was copied to a disaster-recovery location, the deletion job must carry the same tenant, document, and retention identifiers to that location rather than treating the primary bucket as the whole system. During a restore, write the recovered data to an isolated workspace first, record the checksum and source backup ID, and require an explicit promotion step. This is slower than handing a database file to an operator, but it makes the dangerous transitions visible: cross-tenant reads, expired access, partial deletion, and unverified recovery. Those are the boundaries I want in tests and logs before debating which storage engine has the nicer command line.

The failure boundaries matter more than the API vocabulary. A self-hosted deployment gives the team control of its disks and network, but the team also owns disk failure, capacity planning, upgrades, access logs, replication, and the second site. Managed cloud storage reduces that operational surface, while leaving policy, tenant authorization, key naming, restore verification, and provider dependency in the application design. Neither option supplies a retention policy by magic.

Here is the comparison I would put in an architecture review. “Cheapest” means total operating burden over the retention period, not just bytes on a monthly invoice.

| Choice | Tenant-isolation work | Operational boundary | Restore and deletion trade-off |
| --- | --- | --- | --- |
| Self-hosted MinIO | Enforce bucket and prefix policy, network boundaries, and operator separation yourself | Your team runs disks, upgrades, monitoring, replication, and recovery | Good control of placement; a missing offsite copy or weak operator boundary can defeat the design |
| Cloud object storage | Configure identities, private buckets, per-tenant prefixes or buckets, and audit policy | The provider runs the storage service; your team still owns application policy and restore tests | Faster first deployment; provider retention, region, egress, and deletion semantics need review |
| Local filesystem | Implement every boundary in the application and host | The host, filesystem, and backup system are all your responsibility | Simple code path, poor isolation and disaster recovery unless more infrastructure is added |

That last option is useful as a staging fixture, not as the default archive for signed legal or account documents. The storage system should be replaceable, but the tenant policy should not be hidden inside a vendor SDK.

## How do tests evaluate tenant boundaries, object storage, signed URLs, and restore scripts?

Start with an object key that carries identity without trusting the key alone: `tenant/{tenant_id}/documents/{document_id}/backup/{backup_id}.bin`. The authorization check must compare the authenticated tenant with the stored object metadata or a database record before it constructs a key. A guessed prefix is not authorization.

Names are not policies.

For a deletion deadline, store the deadline in the application database and copy it into a job queue. The worker should mark the object as pending deletion, delete the object, verify the resulting absence through the storage API where that API permits it, and record the attempt. If another retained copy exists, the record needs to name it. Otherwise the word “deleted” may describe one bucket while a replica, export, or operator snapshot still contains the document.

The signed-link path is deliberately boring. The server authenticates the requester, checks tenant ownership and deadline state, then asks the object store for a link limited to one key and a short expiration. The browser receives only that link. It should never receive the long-lived application credential.

```python
import os
from datetime import datetime, timezone

import requests


STORAGE_URL = os.environ["STORAGE_URL"]
SERVICE_TOKEN = os.environ["STORAGE_SERVICE_TOKEN"]


def object_key(tenant_id, document_id, backup_id):
    values = (tenant_id, document_id, backup_id)
    if any(not value or "/" in value for value in values):
        raise ValueError("identifiers must be non-empty path-safe values")
    return f"tenant/{tenant_id}/documents/{document_id}/backup/{backup_id}.bin"


def make_restore_url(tenant, document, backup, deadline):
    if tenant != document["tenant_id"]:
        raise PermissionError("tenant boundary rejected")
    if datetime.now(timezone.utc) >= deadline:
        raise PermissionError("retention deadline has passed")

    key = object_key(tenant, document["id"], backup["id"])
    response = requests.post(
        f"{STORAGE_URL}/objects/presign",
        headers={"Authorization": f"Bearer {SERVICE_TOKEN}"},
        json={"key": key, "method": "GET", "expires_seconds": 300},
        timeout=15,
    )
    response.raise_for_status()
    return response.json()["url"]
```

The endpoint above is an interface boundary, not a claim about one provider's route. In production, pin the adapter to the selected storage API and test that the returned link cannot read a sibling tenant's key. A 403 is a useful expected result for that negative test. So is an expired link. Don't turn either test into a reason to make the bucket public.

Upload progress is a client concern, not an authorization mechanism. If a browser uploads directly, use a narrowly scoped signed request and validate the resulting object on the server before accepting the backup record. XMLHttpRequest exposes upload progress events, but progress reaching 100 percent says nothing about checksum validation, tenant ownership, or retention state.

## How do restore scripts implement the recovery contract?

The first restore should be a repeatable job, not a console exercise. Select one backup by tenant, document, and backup ID; download it to a temporary location; verify its checksum and format; restore into an isolated database or workspace; then compare a small set of expected records. Keep the original object untouched until the verification completes.

```python
import hashlib
from pathlib import Path

import requests


def download_and_verify(url, destination, expected_sha256):
    digest = hashlib.sha256()
    path = Path(destination)
    with requests.get(url, stream=True, timeout=60) as response:
        response.raise_for_status()
        with path.open("wb") as output:
            for chunk in response.iter_content(chunk_size=1024 * 1024):
                if chunk:
                    output.write(chunk)
                    digest.update(chunk)

    actual_sha256 = digest.hexdigest()
    if actual_sha256 != expected_sha256:
        path.unlink(missing_ok=True)
        raise ValueError("backup checksum mismatch")
    return path
```

The script needs bounded timeouts, structured logs, and a retry policy that distinguishes a transient rate limit from an authorization failure. Retry a request only when the operation is safe to repeat, and use an idempotency record for uploads so a network timeout does not create an untracked second backup. I would alert on age of the last successful restore test, not merely on the number of successful upload calls.

One short test is worth more than a page of setup instructions. Create two tenants, upload one document for each, request a link as each tenant, attempt the cross-tenant read, expire one link, and run the deletion worker after the deadline. Then simulate a missing object and a bad checksum. Your mileage may vary with the backup format; the test contract should not.

## Failure boundaries and rejected defaults

The catch is operational ownership. Self-hosted MinIO is not suitable when nobody can watch disk capacity, replace failed hardware, maintain a second copy, or rehearse a restore. Cloud object storage is not suitable when the service's region, retention controls, legal hold requirements, egress model, or provider dependency violate the deployment constraints. A single bucket is not suitable when one compromised application credential can reach every tenant without an independent policy boundary.

I am not sure that a raw price comparison can resolve the decision without workload size, request volume, retention duration, egress, and the cost of on-call time. The AWS S3 pricing page is a reminder that object storage billing has multiple dimensions; a small byte total does not describe the restore bill or the engineering effort around self-hosting.

Choose self-hosting when control of the storage location is a requirement and the operational team already exists. Choose a managed service when reducing infrastructure ownership is the requirement. In either case, keep the adapter small, make the tenant check explicit, and make the restore test a release or operations artifact. That is the part that survives a provider change.

## References

- https://aws.amazon.com/s3/pricing/
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
