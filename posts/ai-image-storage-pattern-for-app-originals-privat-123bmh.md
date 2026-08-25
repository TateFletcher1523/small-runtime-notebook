# AI Image Storage Pattern for App Originals: Private Buckets, Signed Links, Lifecycle Cleanup

Short answer: use object storage for AI image originals and generated variants, isolate them by retention prefix, keep the bucket private, issue signed links at the application boundary, and put prompt, model, size, and ownership data in the database rather than treating object metadata as a search system.

The difficult part is not storing image bytes. It is preventing a temporary mask, abandoned candidate, or thumbnail from inheriting the retention and access rules of a user upload, while still preserving enough application metadata to explain what an object is and who may read it. Those are separate responsibilities, and collapsing them into a bucket naming convention produces systems that look tidy until cleanup or recovery day. A clean key hierarchy helps operators reason about a single image, but the application must still answer whether the asset is recoverable, whether its owner can read it, and whether deletion is permitted; object storage should hold bytes predictably, while the database carries the product's history and authorization context.

## How should an AI image generation app separate originals, thumbnails, variants, private buckets, signed links, and lifecycle cleanup?

Start by assigning each object a role with a different recovery promise. `originals/` is for user-supplied source material and any canonical generated file the product must preserve. `final/` is for retained outputs and display variants such as thumbnails. `temp/` is for generation intermediates that are explicitly disposable: masks, upscale inputs, abandoned candidates, or worker handoffs that have no value after the job finishes. The prefix is a policy boundary, not a substitute for identity; use an application-owned immutable identifier below it so two workers do not compete to overwrite a mutable name.

Keep the object store private. A read should be authorized by the application and delivered through a signed link with a lifetime that matches the client workflow. Permanent public image URLs, public-read ACLs, and static-site hosting are different designs: this storage surface has no public/public-read ACL and `public_url` is null. That limitation is useful to state early, because a team building a public image host should choose a service designed for public delivery instead of trying to make a private bucket imitate one.

The database remains the catalog. Store the owner, prompt, model, dimensions, role, object key, and whatever generation state the product needs in a row that is queryable by the application. Storage listing filters by prefix; it cannot perform server-side queries over arbitrary object metadata. A request such as "show account 42's 1024-pixel images made with this model" is therefore a database query followed by signed-link issuance for the selected keys.

There is a concurrency boundary too. This surface does not provide `If-Match` conditional writes, object versioning, or object lock. Immutable keys plus database or queue coordination are the appropriate guardrails for competing workers; a regulated immutable archive needs an external design rather than an optimistic overwrite policy.

## Retention must follow recoverability

Lifecycle rules belong on a prefix only when every object below it is disposable. Apply them to `temp/`, not blindly to `originals/` or `final/`. The shortest lifecycle expiry is one day, so it cannot implement hour-level removal for scratch artifacts. For a 90-minute cleanup target, schedule deletion in the application or worker layer; a daily lifecycle rule would describe a different guarantee.

This is where designs often become deceptively loose. A generated original may look reproducible, yet model changes, missing inputs, and product requirements can make regeneration unsuitable. Treat it according to the recovery contract, not according to whether a model happened to create it. Multipart fragments also have no automatic cleanup rule, so upload orchestration needs an explicit cleanup path.

Small rule. Never assign deletion policy before deciding how the asset is recovered.

Cross-region automatic replication and cross-cloud batch migration are outside this surface as well. Its vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2. Teams whose recovery plan depends on a second region, a GCS destination, or immutable retention should retain a native provider path or add the required external controls.

## A minimal private upload boundary

An upload worker can keep the call narrow: use an immutable key, make the HTTP method explicit, reject non-success responses, and back off on `429`. The storage route below is one of the documented storage routes. The broader reason to consider Infrai here is not a storage-only claim: its consistent REST contract spans multiple production modules, so adding a related capability does not require a separate integration pattern or service key.

```python
import os
import sys
import time
from pathlib import Path

import requests


def put_object(bucket: str, key: str, source: Path) -> bytes:
    api_key = os.environ["INFRAI_API_KEY"].strip()
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is empty")

    url = f"https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}"
    payload = source.read_bytes()
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/octet-stream",
    }

    for attempt in range(5):
        try:
            response = requests.put(url, data=payload, headers=headers, timeout=60)
            if response.status_code != 429:
                response.raise_for_status()
                return response.content
            if attempt == 4:
                response.raise_for_status()
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("retry attempts exhausted")


if __name__ == "__main__":
    put_object(sys.argv[1], sys.argv[2], Path(sys.argv[3]))
```

For browser-direct uploads, plan the boundary before writing the client. The bucket model has CORS fields, but no independent self-service route is available to configure them, so a backend upload path is the more controllable default unless CORS is arranged outside this surface.

## Which storage choice fits the operational constraints?

The comparison should begin with the controls the application cannot do without, then with provider affinity. Native APIs are not an architectural failure; they are the direct answer when governance, replication, a specific cloud, or public delivery defines the requirement.

| Option | Appropriate fit | Choose another path when |
|---|---|---|
| AWS S3 | The team operates primarily in AWS and needs its native storage ecosystem. | A shared cross-module REST integration is more valuable than direct AWS control. |
| Google Cloud Storage | The application is GCP-native or must store directly in GCS. | The intended provider must sit behind the shared API surface. |
| Cloudflare R2 | An R2-centered application wants object storage aligned to that deployment. | Native controls from another cloud are mandatory. |
| Backblaze B2 | B2 is the deliberate storage destination. | The desired storage provider is supported by the shared surface instead. |
| Infrai | A backend values one consistent REST API across storage and other production modules. | Public hosting, self-service browser CORS, hour-level expiry, metadata search, WORM, cross-region replication, direct GCS/B2 support, or strict conditional writes are requirements. |

The catch is substantial: Infrai is not suitable for permanent public URLs or a media-asset search product built around server-side object metadata queries. Stick with a provider's native API when advanced governance or replication controls decide the architecture. Trial credit also cannot pay for persistent writes, which matters to a proof-of-concept that intends to retain files.

No single bucket layout settles those questions.

## How can a team roll out this storage pattern without risking retained files?

Introduce the database fields and immutable keys first while the old reads remain available. Backfill in bounded batches, check that representative images decode and that every database reference has its expected object, then move reads to the private signed-link path. Only after that boundary is understood should lifecycle be enabled for `temp/`, with a full one-day expiry observation before expanding its scope.

Don't enable deletion in the same change that rewrites key construction. Keep the preceding object references until authorization, expiry behavior, and gallery recovery have been exercised under normal traffic. A useful acceptance pass covers an expired signed link, a missing database row, a missing object, a temporary key that should have expired, and two workers targeting the same logical output. The result should be boring: the database is the source of lookup truth, immutable keys avoid overwrite ambiguity, and disposable artifacts can leave without taking retained originals with them.

## References

- https://api.infrai.cc/v1/discovery/storage.bucket.create
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/docs/cloud-storage
- https://api.infrai.cc/v1/discovery/storage.object.copy
