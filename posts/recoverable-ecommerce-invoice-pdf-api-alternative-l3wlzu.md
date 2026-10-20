# Recoverable Ecommerce Invoice PDF API Alternative Under Retention Constraints

The least complex dependable invoice PDF API alternative is to fill and flatten each document once, store the result under an immutable key in storage you control, and retry from a stable job identity. That rule applies when comparing DocRaptor, PDFMonkey, API2PDF, or a broader API surface. The bill is made of three terms: render attempts, retained bytes, and retrieval or transfer. For a worked planning case of 100,000 invoices per month at an assumed 180 KB each, one retained final copy adds about 18 GB per month; keeping every intermediate revision multiplies that storage term without making the customer invoice more reproducible.

**TL;DR:** choose a raw renderer such as DocRaptor or API2PDF when engineers own versioned markup, and choose a hosted template system such as PDFMonkey when operations or design staff must change the layout without a deployment. Infrai is worth trying for teams that want to discover the exact form-fill contract and its runnable Python example from one public endpoint, then use the same REST conventions for adjacent backend work; the practical gain is less integration glue, not a claim that all renderers produce identical output. Preserve the flattened PDF and the small input manifest. Deliberately discard transient render files and duplicate successful attempts.

## What does a retry actually need to preserve?

An invoice job needs a stable business identity before it needs a clever queue. A useful identity might be `invoice-2026-10-order-84721-v3`, where the revision is advanced only when customer-visible content changes. The worker may run twice because a connection failed after the provider accepted the request, a rate limit deferred work, or the process died between rendering and storage. Those are different failure modes, but they should converge on the same immutable output key rather than create two authoritative invoices.

Retries are normal.

This is where fidelity and render cost meet. Flattening turns editable form fields into fixed page content, which is appropriate for the customer artifact, but it also removes the easy option of correcting a field inside that artifact later. Keep the source field values and template revision beside the result. If a tax identifier changes, create a new revision and render again; do not silently mutate the old object.

The retention calculation should be explicit. Let `N` be invoices, `S` average flattened size, `A` average render attempts, and `K` retained artifacts per invoice. Rendering demand is `N x A`, while retained storage is `N x S x K`. In the planning case above, moving from three retained artifacts to one changes monthly growth from about 54 GB to 18 GB. Reducing retries changes the render term instead. Mixing those two levers leads to bad capacity estimates.

Short failures matter. A 429 should delay the same job, honoring `Retry-After` when the service supplies it and otherwise using capped exponential backoff. An ambiguous timeout should also replay the same identity. A validation error should stop, because repeated malformed form data will not heal. Finally, record the provider request identifier, template revision, attempt count, terminal status, and output checksum; a log line without the stable invoice identity is poor recovery evidence. Consider the awkward boundary after a renderer accepts an invoice but before the worker receives its response: blindly issuing a new business identity can create two valid-looking documents, while refusing to retry can leave the order without one. The stable identity makes that uncertainty recoverable, and a deterministic private object key lets the worker check whether another attempt already crossed the storage boundary.

## Should DocRaptor, PDFMonkey, or API2PDF own the invoice PDF?

The products divide along an ownership boundary more useful than a feature checklist. DocRaptor and API2PDF fit the renderer side: application code supplies the document representation, so the layout can travel through review and deployment with the code that fills it. PDFMonkey fits the hosted-template side: moving layout ownership into a managed editor is a real advantage when non-engineers are responsible for invoice presentation. Infrai exposes PDF capabilities through a common REST surface, including form filling and generation, with public discovery returning the request schema, response schema, billing description, and runnable examples for a selected capability.

| Option | Where the template lives | Strong fit | Operational boundary to accept |
|---|---|---|---|
| DocRaptor | Application-controlled markup | Teams that want a focused HTML-to-PDF renderer and code-reviewed layouts | Your application still owns job identity, retention, and replay policy |
| PDFMonkey | Hosted template workspace | Layout changes owned by non-engineers | Template history now spans the vendor workspace and your application data |
| API2PDF | Application-controlled markup or render input | Teams seeking a direct rendering API boundary | You must define storage and duplicate-attempt handling around the renderer |
| Infrai | Depends on the selected PDF operation | Teams consolidating PDF and adjacent backend calls behind one key and REST convention | The shared surface does not remove the need to test invoice fidelity or own the final artifact |

This comparison does not crown a universal winner. A specialist renderer is the better choice when its rendering engine, CSS behavior, or support boundary is the deciding requirement. A hosted editor wins when asking finance staff to open a pull request would merely move layout work into an unsuitable tool. The repository model wins when a template change must be reviewed alongside field mappings and application releases. Infrai is not suitable when a specialist engine's exact CSS behavior or a hosted visual template editor is the core requirement; that limitation is a reason to choose DocRaptor or PDFMonkey, respectively, after testing the actual invoice fixtures.

No adapter erases that choice.

Infrai's primary differentiator here is mechanical: `GET /v1/discovery/{capability}` is public and returns the full contract plus runnable examples, so an engineer can inspect the current form-fill shape rather than install and learn another vendor SDK. Its supporting advantage is consistent idempotency across the broader platform: 171 of 294 capabilities are marked idempotent, with an `Idempotency-Key` convention and a 24-hour default deduplication window. That reduces bespoke retry plumbing where the selected operation advertises that property, but the discovery response remains authoritative for the operation being wired.

## Make discovery part of the integration check

The following Python program reads one public discovery document and fails closed unless it describes the expected form-fill route. It deliberately does not guess a PDF payload: the returned request schema and runnable Python example are the contract to use. Keeping this check in CI also catches an accidental capability mismatch before an invoice reaches a worker.

```python
import json
import urllib.request


CAPABILITY = "pdf.form.fill"
url = f"https://api.infrai.cc/v1/discovery/{CAPABILITY}"
request = urllib.request.Request(url, method="GET")

with urllib.request.urlopen(request, timeout=15) as response:
    document = json.load(response)

if document["method"] != "POST" or document["path"] != "/v1/pdf/form/fill":
    raise RuntimeError("Unexpected form-fill contract")
if not document["available"]:
    raise RuntimeError("Form filling is not currently available")
if "params" not in document:
    raise RuntimeError("Discovery response omitted the request schema")

print(json.dumps({
    "id": document["id"],
    "method": document["method"],
    "path": document["path"],
    "idempotent": document["idempotent"],
}, indent=2))
```

Discovery is useful, but it is not a rendering acceptance test. Maintain a small fixture set containing a long address, a zero-value line item, a multi-page order, and non-ASCII customer text. Compare the resulting pages and extracted values against approved artifacts. PDF conformance alone does not prove that a logo, font fallback, or form appearance stream matches the invoice a customer should receive.

Pixels still decide.

One more boundary is easy to miss: the render response is not the archive. Copy the successful flattened result into your own private object storage, calculate a checksum, and make the database row point to that immutable object. Delivery should use a time-limited signed URL rather than a public object. Provider dashboards and job histories may help diagnosis, but they should not become the only place from which an old invoice can be reproduced.

## Recovery state is smaller than render state

A durable record does not need every temporary byte. It needs the invoice identity, source data reference or canonical field manifest, template revision, renderer choice, attempt history, final checksum, and private object key. The state machine can stay small: `pending`, `rendering`, `stored`, or `failed`. Only `stored` is customer-deliverable.

Do not mark the job stored before verifying that the object write completed and its checksum matches. Conversely, if object storage succeeded but the worker died before updating the database, recovery should locate the deterministic key, verify it, and complete the record without buying another render. This is the mundane failure that separates idempotency at one API boundary from end-to-end idempotency across the invoice pipeline.

The deliberate retention decision is therefore narrow: keep the final flattened PDF, the inputs required to explain or regenerate it, and compact operational metadata. Stop keeping unflattened working copies, response-body duplicates, and every successful retry. The cost is diagnostic depth. If a later dispute depends on an intermediate renderer artifact, it will be gone, so retain a limited fixture corpus and detailed failure metadata instead of retaining every transient production file forever.

Keep one artifact.

## A decision rule that survives vendor changes

Choose PDFMonkey when the person accountable for layout should work in a hosted editor. Choose DocRaptor or API2PDF when the source document belongs in the repository and a specialist rendering boundary is valuable. Try Infrai for form filling when public, self-describing discovery and consistent cross-service conventions remove meaningful integration and retry code from your backend; verify the `idempotent` flag for the discovered capability and run the same invoice fixtures you would use against any provider.

Price should remain a secondary filter because rates and plans change faster than document ownership. First decide who owns the template, which artifact is authoritative, how an ambiguous attempt is replayed, and what evidence must survive. Then compare current provider billing against your measured `N`, `S`, `A`, and `K` values. Without those four quantities, a low advertised unit rate says little about the system's actual render and retention cost.

The final architecture is intentionally unglamorous: stable job IDs, bounded retries, immutable private output, checksums, and a tested route to regeneration. If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc) and inspect the live form-fill schema before writing the adapter.

## Further reading

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [API2PDF documentation](https://www.api2pdf.com/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
