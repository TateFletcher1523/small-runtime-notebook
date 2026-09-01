# Fintech Tenant Isolation: Delete Old Thumbnails on Image Replace in Object Storage

Short answer: for authenticated fintech reports, do not delete the previous image set during the upload transaction. Write a new generation, commit the tenant-scoped pointer, then queue exact-prefix cleanup; a one-day lifecycle rule is a recovery net, not the replacement protocol.

The important boundary is ownership. A customer request must resolve an image through an authenticated report row containing both `tenant_id` and the active generation. It must never turn a request parameter directly into an object-store prefix. That extra lookup feels fussy until two customers use the same report slug, or a retry arrives after the report has already moved to a new generation.

This is a data-integrity problem before it is a storage-cleanup problem.

## What should a tenant-isolated report replacement guarantee?

For a report image, the system should guarantee three things: readers see one committed generation, cleanup cannot select another tenant's objects, and a worker crash leaves a retryable job rather than an undocumented partial state. “The old files are gone” is a weaker goal and, by itself, a dangerous one.

Use keys whose path encodes an internal tenant identifier and an immutable generation, for example `reports/t-1842/r-907/images/g31/`. Keep the active generation in the database. The prefix is an efficient deletion boundary; it is not an authorization mechanism. Authorization remains a database and application concern.

The replacement sequence is deliberately boring:

1. Authenticate the caller and load the report with its tenant boundary.
2. Allocate a new generation and write the original plus every derived thumbnail beneath that generation.
3. Verify the expected variants and their metadata before changing the active pointer.
4. In one database transaction, update the active generation and insert a cleanup job for the retired generation.
5. Let a worker list the retired prefix, delete complete batches, and retry until the listing is exhausted.

The database transaction is the hand-off. Storage reclamation is asynchronous.

If the upload fails, the old generation remains live. If the worker fails after one batch, the same cleanup job can resume. If a caller retries the replacement, the generation identifier prevents a late cleanup request from guessing which files are current.

## How do prefix listing, batch deletion, and lifecycle differ in this cleanup?

Prefix listing answers “which keys belong to this retired generation?” Batch deletion answers “which of those keys can this worker request together?” A lifecycle rule answers “which objects old enough for this policy may be removed later?” They solve different parts of the problem, so substituting one for another produces a clean-looking but unsafe design.

The one-day minimum matters for thumbnail retention. A lifecycle policy cannot express “delete immediately after this particular image row changes” when its eligibility floor is one day. That delay may be acceptable for abandoned derivatives, but it is unsuitable as the only mechanism when the product promises prompt reclamation or when a tenant's retention contract is shorter.

There is a second trap: listing and deletion are separate observations. A worker can list a key, then the application can publish a newer generation before the delete request runs. The job therefore needs an immutable retired generation, and the worker must refuse a job if that generation is active again. A prefix string alone cannot make that check.

Here is the storage-independent part of a worker. The adapter is intentionally small; its implementation should follow the object-storage API selected by the team, including that API's pagination and batch-size limits.

```python
from dataclasses import dataclass, field
from typing import Protocol


class Store(Protocol):
    def list_page(self, bucket: str, prefix: str, cursor: str | None) -> tuple[list[str], str | None]: ...
    def delete_batch(self, bucket: str, keys: list[str]) -> None: ...


@dataclass
class CleanupJob:
    tenant_id: str
    report_id: str
    retired_generation: str
    active_generation: str
    bucket: str


def clean_retired_generation(store: Store, job: CleanupJob) -> int:
    if job.retired_generation == job.active_generation:
        return 0

    prefix = (
        f"reports/{job.tenant_id}/{job.report_id}/"
        f"images/{job.retired_generation}/"
    )
    cursor = None
    deleted = 0

    while True:
        keys, cursor = store.list_page(job.bucket, prefix, cursor)
        if keys:
            store.delete_batch(job.bucket, keys)
            deleted += len(keys)
        if cursor is None:
            return deleted
```

The `active_generation` check is only useful when it is fresh. In production, read it from the same transactional state that owns the cleanup job, or use a lease/version check that makes stale workers stop. Do not cache tenant authorization in a process and call that isolation.

I would test this worker with two tenants sharing a report identifier, a missing thumbnail, an empty page, a multi-page listing, a crash after each deletion batch, and a cleanup job whose generation has been restored. Those cases expose more than a happy-path assertion that one set disappeared.

## Which failure modes deserve a hard stop?

The most serious failure is a key derived from an untrusted slug. A caller who can influence `tenant_id`, `report_id`, or a prefix can turn cleanup into cross-tenant deletion. Treat those values as identifiers loaded from the authorized report row, and reject path components that are not produced by the application. Imagine a cleanup message for tenant A arriving after a report rename, while a worker has already cached the old slug and a second tenant happens to use that same human-readable name. If the worker rebuilds a prefix from the message rather than loading the immutable report record, its list call can select objects that the authenticated caller never owned. The delete request may be perfectly valid to storage and still be a security incident. That's why the queue should carry a job ID and the worker should resolve the tenant, report, retired generation, and authorization context from durable state before it lists anything.

The next failure is publishing the database pointer before all required variants exist. A report may then contain a committed generation that returns a 404 for its medium thumbnail. The writer should record the generation as pending until the variant set is complete; readers should continue using the prior generation during that interval.

It's a small state machine.

Deletion itself is not a transaction with the database. A timeout can mean that some objects were removed and others were not. Make the operation idempotent, persist the last successful page or a resumable cursor where the selected API permits it, and treat an already-absent object as a successful cleanup outcome. Never mark the job complete after the first page.

For regulated financial data, retention is also a policy decision. NIST's HIPAA Security Rule guidance is not a fintech storage specification, but its control-oriented treatment is a useful reminder that access, audit, integrity, and retention need evidence at the system level. Object versioning can support recovery, yet it can also preserve prior object versions outside the visible key set; deleting current keys is not proof that every retained version disappeared.

The catch is that generation-based cleanup is not suitable when the business requires immutable historical report artifacts or a formal write-once retention control. In that case, keep historical generations under an explicit retention policy and choose storage controls that match the legal requirement; do not quietly reuse a garbage-collection worker as an erasure policy.

## How should a Node.js team roll this out without weakening isolation?

Start with reads. Add a database query that returns the report's tenant and active generation, then make the image-serving path derive every key from that result. Add metrics for missing variants, cross-tenant authorization denials, cleanup age, pages scanned, and deleted keys before enabling deletion.

Next, dual-write new replacements under generations while leaving the old generation untouched. Compare the expected manifest with the objects actually written. Once the read path is generation-aware, enqueue cleanup for a small cohort and inspect audit records containing tenant, report, generation, actor, and job ID.

Keep the deletion worker narrow. It should receive a cleanup-job identifier, not an arbitrary prefix from a queue payload. It should re-check the active generation, process pagination, retry bounded failures, and send exhausted jobs to an operator-visible state. A one-day lifecycle rule can cover orphaned objects after the operational path is proven, but it should not decide which generation is live.

One final rule: deletion must be less powerful than publication. Publication changes the database pointer only after the new manifest is ready; cleanup can remove only a generation that the database still identifies as retired. That asymmetry is what protects an authenticated customer report when retries, duplicate messages, and partial uploads arrive in an unpleasant order.

## References

- https://csrc.nist.gov/pubs/sp/800/66/r2/final
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
