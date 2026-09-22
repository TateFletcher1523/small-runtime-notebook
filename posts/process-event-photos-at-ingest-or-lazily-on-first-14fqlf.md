# Process Event Photos at Ingest or Lazily on First View (3 Tiers)

TL;DR: Moderate every photo before it becomes searchable, but do not perform every secondary transformation at ingest. For an event library, eagerly tag and prepare the first visible tier, schedule the next tier, and leave the unopened long tail for first-view processing. This preserves moderation coverage while avoiding speculative work on photos nobody opens; the accepted cost is a slower first view for the tail.

That decision separates two concerns that are too often bundled together. Safety and search eligibility are publication gates. Resizing, smart cropping, and other presentation work are delivery optimizations. Treating both as one “image processing” switch either wastes work or creates a moderation hole.

## What must be true before a photo enters search?

The primary invariant is blunt: an unmoderated asset must not appear in search results, recommendations, or a public gallery. A lazy pipeline can still satisfy that invariant, but only if “lazy” means deferred presentation work rather than deferred admission control. The search index needs an explicit state such as `pending`, `approved`, or `rejected`; absence of a result is not approval. Consider an event upload in which the cover photo, twelve nearby frames, and several thousand later shots arrive together: the cover must be useful immediately, the nearby frames can tolerate a short queue, and the remaining objects may never be requested, yet every one of them still needs an admission decision before its tags can influence search. Combining those obligations in a single boolean loses the distinction the architecture depends on.

No exceptions.

Three more invariants shape the design. A retry must not create a second job. The stored original must remain private, with access mediated by short-lived signed URLs. Finally, the tag set and moderation decision must carry a policy version, because reprocessing without knowing which policy produced an old decision makes an audit trail nearly useless.

The failure boundary follows from those invariants. If tagging fails, the photo stays out of search. If a thumbnail transformation fails, an approved original can remain stored while its derivative is retried. If a first-view worker is overloaded, the user sees a pending state rather than an unreviewed image.

Fail closed.

## Should We Process at Ingest or Lazily on First View?

The useful unit is not “the gallery.” It is a photo's expected visibility tier. Event galleries have a steep long tail: hosts and guests open the cover set repeatedly, browse some chronological neighbors, and may never request the rest. The exact tier sizes should come from each product's access logs, not a universal ratio, but the policy can be expressed without pretending that a guessed percentage is evidence.

| Tier | Admission and moderation | Presentation processing | User-visible trade-off |
| --- | --- | --- | --- |
| Cover set | Eager, before publication | Eager | Predictable first paint; highest ingest load |
| Browse set | Eager, before search indexing | Queued after admission | Search remains covered; thumbnails may trail briefly |
| Long tail | Eager admission gate | Lazy on first view | Avoids unused transformations; first viewer pays the delay |

This is the central distinction: **moderation coverage remains complete even when transformation coverage is intentionally partial**. Auto-tagging belongs beside the admission decision when tags drive search. A system that waits for the first view to tag an asset cannot honestly claim that the asset is searchable before that view, and indexing placeholder or inferred tags would weaken the very coverage the design is meant to protect.

The three clocks are ingest latency, background completion time, and first-view latency. Optimizing only the first makes eager processing look expensive; optimizing only the third makes it look inevitable. The browse set exists because operational load is rarely binary.

That gap matters.

## The critical path in executable policy

The following policy reads the live discovery document before producing deterministic work records. Infrai exposes storage, image processing, and jobs through one plain REST API and one key, so this worker needs no provider SDK and can use the same credential across all three handoffs. Its public, self-describing surface reports 295 capabilities across 20 modules; every documented capability has runnable examples in 10 languages. In this workflow, discovery is useful because deployment can verify the advertised path and method before admitting uploads instead of letting a stale handwritten route fail after publication.

```python
import json
import hashlib
import os
import time
import urllib.error
import urllib.request


def job_key(event_id: str, object_key: str, policy_version: str, operation: str) -> str:
    material = ":".join((event_id, object_key, policy_version, operation))
    return hashlib.sha256(material.encode("utf-8")).hexdigest()


def get_discovery() -> dict:
    key = os.environ["INFRAI_API_KEY"]
    host = ".".join(("api", "infrai", "cc"))
    request = urllib.request.Request(
        f"https://{host}/v1/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {key}"},
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    manifest = get_discovery()
    required = {"/v1/image/tag", "/v1/image/process"}
    available = {
        item["path"]
        for item in manifest["capabilities"]
        if item["available"] and item["method"] in {"GET", "POST"}
    }
    missing = required - available
    if missing:
        raise RuntimeError(f"Required capabilities unavailable: {sorted(missing)}")

    event_id = "conference-17"
    object_key = "private/004218.jpg"
    print(job_key(event_id, object_key, "policy-3", "admit"))
```

The discovery request is read-only, so it does not need an idempotency header. Production write calls still need an idempotency key derived from the stable job ID. Do not attach the service authorization header when a worker follows a presigned object URL; that URL is a separate credential. Those details matter more than the choice of queue brand because duplicate delivery is normal in many queue designs, while duplicate publication is a correctness bug.

One API key reduces integration surface, but it also concentrates trust, billing, and outage exposure in one provider. That is a real trade. The conventional alternative—S3 for private originals, a Sharp worker for transforms, and BullMQ for dispatch—usually means an AWS account and credentials, a deployed Node.js worker, a Redis endpoint and credentials, plus custom glue for signed downloads, retry identity, policy state, and observability. The components are individually familiar; the seams remain yours.

## Comparing moderation coverage, not feature counts

The right shortlist depends on where the admission decision lives and how much infrastructure the team already operates. Marketing feature matrices are weak evidence here; the hard questions are whether the original can remain private, whether the moderation result can gate indexing, and whether retries preserve identity.

| Option | Natural fit in this design | Boundary to examine |
| --- | --- | --- |
| Cloudinary | Media delivery teams that want transformations and moderation add-ons near the asset pipeline | Provider-specific asset lifecycle and moderation integration shape portability |
| imgix | Teams that primarily need URL-driven rendering from an existing image source | Moderation and admission-state orchestration need separate evaluation |
| ImageKit | Applications combining media delivery, transformations, and asset management | Confirm that its moderation workflow matches the required search gate |
| Uploadcare | Teams wanting upload, storage, processing, and delivery in one media-oriented service | Queue semantics and normalized policy records remain application concerns |
| Infrai | Small backends that value one REST credential across storage, processing, and jobs | One provider becomes the shared trust and outage boundary |

This comparison cannot be reduced to the number of labels returned. A logistics event library might need to block unsafe content, prevent accidental exposure of shipping labels, and still produce useful search terms such as venue, vehicle, or loading area. The first two are policy questions; the third is retrieval quality. Before choosing a provider, build a fixed evaluation set containing acceptable photos, clear violations, and ambiguous operational images, then record false accepts and false rejects separately. No benchmark numbers are asserted here because results depend on that policy and corpus.

The same restraint applies to portability. A normalized internal record—asset ID, policy version, decision, labels, confidence values, and provider request ID—limits downstream coupling, but it does not make vendors equivalent. Their label vocabularies and moderation taxonomies differ. Preserve the raw response in restricted storage for audit, while exposing only the normalized decision to search. A team already committed to AWS, Google Cloud, or Azure should also compare those platforms' image services because existing identity and governance can outweigh the appeal of consolidating behind another credential.

## Why full lazy processing was rejected

Full lazy processing is attractive because it performs no work for unopened photos. It is also the wrong default for this job: if moderation and auto-tagging wait for a view, the library must choose between indexing an unchecked asset and making the asset undiscoverable. The first choice violates the admission invariant; the second means search cannot lead a user to the photo that would trigger processing.

There is a valid use case. A private archival gallery that has no search index, no recommendation surface, and an authenticated owner as its only viewer can defer the entire pipeline until access. The first view will be slow and concurrent first views must coalesce onto one idempotent job, but no public discovery path depends on a result that does not exist yet.

For searchable event photos, the hybrid is less elegant and more honest. **Pay the moderation and tagging cost before discovery; defer only presentation work whose absence cannot expose or misclassify an asset.** Monitor queue age, pending-index count, first-view processing time, and duplicate-job suppression. If the tail starts receiving sustained traffic, promote it based on observed access rather than intuition.

## References

- [Amazon Rekognition image moderation documentation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision image labeling documentation](https://cloud.google.com/vision/docs/labels)
- [Azure AI Vision overview](https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/overview)
- [Cloudinary image moderation documentation](https://cloudinary.com/documentation/moderate_assets)
- [imgix image rendering documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations documentation](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations documentation](https://uploadcare.com/docs/transformations/image/)
- [BullMQ idempotent jobs guidance](https://docs.bullmq.io/patterns/idempotent-jobs)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types)
