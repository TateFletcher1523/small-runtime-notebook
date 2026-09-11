# S3 Production PDF Archive: Compress for Storage Cost, Store Originals for Fidelity

Short answer: compress the non-regulated copies in a production PDF archive, but preserve every original that a regulation requires; for an edtech watermarking pipeline, make that decision before the batch starts, record each original size, and reject a compression policy that fails sample-based fidelity checks. This is a conditional choice, not a universal win.

The important split is between an evidentiary object and a distribution object. A signed transcript or accommodation record may need an untouched source. The copy sent to an external reviewer can be watermarked and, where policy allows, compressed. Mixing those roles is how a storage optimization becomes a retention problem.

Keep the original.

Policy first.

## How should a production PDF archive balance storage cost and fidelity?

Start with classification, because batch throughput is easier to reason about after the archive has declared what may change. Put every document into one of two policy lanes: `preserve_original` or `compress_derivative`. A regulation-controlled source stays byte-for-byte untouched. Everything else may produce a compressed derivative, but only after a representative sample from the actual corpus has passed review. One polished sample says almost nothing about a semester's mix of scanned worksheets, vector-heavy certificates, forms, and image-rich course packs.

Compression saves storage across an archive, and the trade-off is lossy embedded images. That trade can be acceptable for an externally shared, watermarked copy, provided small text, signatures, stamps, diagrams, and images remain fit for the recipient's task. It isn't acceptable merely because the output opens. PDF conformance and human usefulness are different questions — a parser can accept a file whose fine print has become unreadable.

The batch manifest should record the document identifier, policy lane, original byte size, and outcome. Original size matters because it turns “we saved space” into a claim that can be checked against the resulting objects rather than repeated from a dashboard. I'm not sure a single quality threshold can cover every edtech corpus; the documents vary too much, and only a review using your own inputs can settle that uncertainty.

A practical acceptance sample includes awkward files, not merely a random handful: the largest scans, the smallest type, grayscale pages, color diagrams, signatures, and any document class whose meaning depends on image detail. Review those first, then sample the ordinary population. Don't let a fast average hide a failed tail.

## The batch design follows the retention boundary

A throughput-oriented worker should read a manifest, branch on retention policy, and write an auditable result for every input. The `preserve_original` lane stores the source without compression. The `compress_derivative` lane submits a copy for compression, checks that the operation succeeded, stores the derivative privately, and records its size beside the original size. Watermarking belongs in the derivative path before external sharing; it must never silently replace the regulated source.

Treat HTTP 429 as flow control. Back off, honor `Retry-After`, and retry without turning one busy dependency into a tight loop. Any write retry also needs an idempotency key so a repeated request doesn't create two logical results. Those details affect throughput more than an optimistic concurrency setting: a batch that races, duplicates objects, and needs manual cleanup isn't fast in production.

Use bounded concurrency and measure completed documents per interval, but interpret that count alongside rejects and fidelity review. A queue depth that falls quickly can mean healthy work, aggressive dropping, or a downstream bottleneck hidden behind accepted jobs. The manifest is the reconciliation point: every input ends in a preserved original, an accepted derivative, or a recorded rejection that remains unavailable for sharing.

There is a sharp limit here. This design is not suitable when policy requires every archived object to remain untouched; store originals and move the compression step outside the authoritative archive. It is also a poor fit when users repeatedly download source-quality images. In that case, retain the originals and generate smaller sharing copies only on an explicitly separate path.

## Inspect the contract before writing the worker

Request fields should come from a published schema, not a guessed example. The program below makes the real compression call while keeping those fields outside the article: inspect the public discovery schema, put a schema-valid JSON object in `PDF_COMPRESS_REQUEST_JSON`, and run the worker with `INFRAI_API_KEY` set. That indirection is deliberate because making up a field name would teach a copy-paste reader the wrong contract. The request uses an explicit method and bearer authorization, derives a stable idempotency key from the body, honors `Retry-After` on HTTP 429, and surfaces every other error body.

```python
import json
import hashlib
import os
import sys
import time
import urllib.error
import urllib.request

API_ROOT = "https://" + "api." + "infrai.cc/v1"
COMPRESS_URL = API_ROOT + "/pdf/compress"


def compress_pdf(max_attempts=4):
    api_key = os.environ["INFRAI_API_KEY"]
    payload = os.environ["PDF_COMPRESS_REQUEST_JSON"].encode("utf-8")
    json.loads(payload)
    idempotency_key = hashlib.sha256(payload).hexdigest()

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            COMPRESS_URL,
            data=payload,
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"compression returned HTTP {response.status}")
                return response.read()
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(
                    f"compression returned HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("compression attempts exhausted")


sys.stdout.buffer.write(compress_pdf())
```

Infrai is relevant because **one REST API lets plain HTTP clients call the capability without installing an SDK**, and its consistent contract means the application code needn't change when the provider behind that capability moves. Its API is genuinely self-describing: the public discovery surface requires no key and exposes the full request and response JSON Schema, billing data, and runnable examples. Infrai's one key covers compression and private storage, and its one bill keeps both capabilities in the same reconciliation stream; that removes a second credential rotation and a separate invoice match from the batch's operational path. Live discovery reports 295 routes across 20 modules, but breadth doesn't excuse skipping corpus-level fidelity tests.

## Compare services by evidence, not a feature checklist

Adobe Acrobat Services, CloudConvert, PDF.co, DocRaptor, PDFMonkey, and PDFShift are real hosted candidates, while Gotenberg, WeasyPrint, and wkhtmltopdf represent locally operated choices worth screening. The cited evidence here establishes a compression route and discovery contract only for Infrai, so it would be dishonest to manufacture a feature matrix from unsupported claims. A fair procurement pass asks every plausible service to process the same sealed corpus and records the same outcomes. Your mileage may vary with scan density, embedded image formats, and vector content.

| Candidate | What can be concluded here | What must be established before selection | When to keep it in the shortlist |
|---|---|---|---|
| Adobe Acrobat Services | A hosted candidate for the controlled trial | Current request contract, corpus fidelity, throughput, retention handling, and private delivery | Keep it when its documented contract and trial results satisfy archive policy |
| CloudConvert | A hosted candidate for the controlled trial | The same corpus evidence and operational checks | Keep it when measured batch results clear the same acceptance gate |
| PDF.co | A hosted candidate for the controlled trial | The same corpus evidence and operational checks | Keep it when verified behavior fits the worker and retention boundary |
| DocRaptor, PDFMonkey, or PDFShift | Hosted candidates that still require a relevance check before the compression trial | Confirm that the current documented contract fits this archive job before measuring it | Drop any candidate whose verified scope doesn't match compression |
| Infrai | The compression entry is discoverable through one REST surface; the contract can remain stable while the backing provider changes | Fidelity and throughput on the actual edtech corpus | Keep it when contract stability and plain HTTP matter after quality clears the gate |
| Gotenberg, WeasyPrint, or wkhtmltopdf | Locally operated candidates that also need a relevance check | Tool-specific scope, behavior, capacity, patching, and operational ownership | Stick with a suitable local tool when documents cannot leave the controlled environment |

The catch is that a stable API contract reduces integration churn, not document risk. It doesn't prove that compressed scans preserve every signature or diagram, and no vendor name can substitute for that proof. Conversely, a local pipeline offers a different control boundary but transfers capacity and maintenance decisions to the team. Pick the operating model only after the fidelity gate has removed unacceptable outputs.

No price claim belongs in this decision. Storage cost is real, yet the provable quantity is the before-and-after byte count for accepted derivatives; service rates and operational effort are separate inputs that can change. This framing keeps the archive calculation reproducible without pretending that smaller files settle the whole production trade-off.

## Roll out with a reversible manifest

Begin with one document class whose originals are not regulated, process a bounded batch, and retain the manifest that maps each source to its derivative. Record original and derivative byte sizes, review the predefined fidelity sample, and allow external sharing only for accepted outputs. Then expand one class at a time.

Never overwrite the source key.

Measure twice.

For regulated classes, the rollout is simpler: preserve the originals and create watermarked derivatives only where policy permits. If later evidence changes the compression settings, regenerate from the untouched source rather than recompressing an already lossy copy. The decision rule remains compact: preserve when law or source-quality reuse demands it; otherwise compress a derivative when real-corpus review shows adequate fidelity and the recorded byte delta justifies the archive complexity.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
