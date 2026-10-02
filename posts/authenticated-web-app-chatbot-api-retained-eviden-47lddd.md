# Authenticated Web App Chatbot API — Retained Evidence for Accurate Invoice Extraction

TL;DR: For an authenticated web app chatbot that extracts fields from gaming supplier invoices, the least complex backend API is a thin service that owns authorization, emits a small event vocabulary, validates one final JSON object, and stores the invoice, final result, validation report, and usage counters. Do not retain every streamed fragment by default. The dominant storage term is the data copied per extraction multiplied by retention time and replica count; changing that term matters more than swapping client libraries or choosing an SDK alternative.

A streaming SDK can make the demo pleasant, but it cannot decide which studio may read an invoice, whether `total` is a number or a persuasive sentence, or how much evidence must survive a dispute. Those are backend and data-contract decisions. **Choose candidates by structured-output correctness under interruption and replay, not by the elegance of their happy-path stream helper.**

## What is the storage bill actually made of?

Start with bytes, because a retention design without a byte model is wishful thinking. For one extraction, let `D` be the original invoice bytes, `T` the extracted text or document representation, `S` all streamed fragments and transport envelopes, `R` the final structured result plus validation report, and `M` the request metadata and usage counters. With `N` extractions retained for `d` days and `k` durable copies, the rough retained volume is:

`N × d × k × (D + T + S + R + M)`

This is a capacity model, not a price claim. Measure each term from production-shaped fixtures before attaching any currency to it. The trap is `S`: if the service appends a database row for every fragment, repeats the accumulated answer in each event, or stores request and response bodies in ordinary logs, a transient presentation mechanism becomes durable data. Retrying a disconnected request can duplicate it again.

Consider an accounting exercise rather than a benchmark. Suppose an internal test corpus contains 10,000 supplier invoices, each fixture is 400 KB, its normalized text is 80 KB, its complete stream trace is 120 KB, and its final JSON result plus validation record is 4 KB. Those are hypothetical planning inputs, not measurements. At one durable copy, the terms are 4.0 GB, 0.8 GB, 1.2 GB, and 0.04 GB respectively. Removing stream traces from routine retention cuts the modeled corpus from 6.04 GB to 4.84 GB; deleting the source invoices would move more bytes, but it would also destroy the strongest evidence for checking extraction errors.

That distinction controls the design. Keep source evidence according to the business retention rule. Keep compact final results and validation outcomes for audit and regression selection. Retain complete stream traces only for a small, access-controlled diagnostic sample with an explicit expiry. Logs should carry identifiers, event counts, durations, result status, and byte counts, not invoice bodies or fragment text.

| Retained artifact | Correctness value | Failure mode if kept indiscriminately | Default decision |
|---|---|---|---|
| Original invoice | Replays extraction against the actual evidence | Sensitive supplier data outlives its purpose | Retain by business policy, encrypted and access-controlled |
| Normalized text | Helps isolate parsing from model behavior | Becomes a second searchable copy of the invoice | Retain only when replay needs it |
| Stream fragments | Diagnoses ordering and truncation | High object or row count; accidental content logging | Sample briefly, then expire |
| Final JSON and validator report | Supports downstream reconciliation | Invalid output may be mistaken for accepted output | Retain with schema version and status |
| Counters and timings | Supports capacity and latency analysis | Labels can create high-cardinality telemetry | Retain without document content |

The material change is straightforward: stream to the browser, but persist the accepted result rather than the delivery transcript.

Fragments are not records.

## What should an authenticated web app chatbot API accept from a stream?

HTTP success and a clean end-of-stream marker say nothing about whether the extracted object is usable. A supplier invoice can contain a purchase-order identifier, invoice number, currency, subtotal, tax, total, and line items; the dangerous errors are syntactically tidy. A string may occupy a numeric field. The currency may be missing. Two reconnects may commit two records. A plausible number may not be traceable to the source at all.

Treat the streamed text as provisional display data. The backend accepts exactly one terminal candidate, parses it, checks a versioned schema, applies deterministic domain invariants, and commits it under an idempotency key scoped to the authenticated tenant. No partial fragment reaches the ledger or approval workflow.

The browser should receive a deliberately small protocol. An `ack` event associates the request with a server-generated operation identifier; zero or more `delta` events update presentation; one `result` event carries the validated object; an `error` event carries a stable machine code; and an `end` event closes the attempt. Sequence numbers let the client reject duplicates and notice gaps. They do not make replay safe by themselves. That requires a server-side uniqueness rule for the tenant and idempotency key.

Backpressure and cancellation also belong at this boundary. If the browser is slow or disappears, the server must stop buffering without limit, cancel work where cancellation is supported, and record whether a final result was committed. A timeout after several visible fragments is still a failed extraction unless a validated terminal object already exists.

## Validate once, commit once

A candidate-specific SDK should sit behind an adapter that returns ordinary events and a terminal object. The core service should not expose a provider's fragment classes, exception hierarchy, or authentication model to the browser. This focused Python example shows the more important half of the boundary: strict parsing, domain checks, and an atomic idempotent commit. The in-memory repository is illustrative; a deployed repository needs a transactional uniqueness constraint with the same key.

```python
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
import json
from typing import Any


REQUIRED_FIELDS = {
    "invoice_number",
    "currency",
    "subtotal",
    "tax",
    "total",
}


@dataclass(frozen=True)
class AcceptedExtraction:
    tenant_id: str
    request_key: str
    schema_version: str
    result: dict[str, Any]


def validate_invoice(raw: str) -> dict[str, Any]:
    value = json.loads(raw)
    if not isinstance(value, dict) or set(value) != REQUIRED_FIELDS:
        raise ValueError("unexpected invoice fields")

    if not isinstance(value["invoice_number"], str):
        raise ValueError("invoice_number must be a string")
    if not isinstance(value["currency"], str) or len(value["currency"]) != 3:
        raise ValueError("currency must be a three-letter code")

    try:
        subtotal = Decimal(str(value["subtotal"]))
        tax = Decimal(str(value["tax"]))
        total = Decimal(str(value["total"]))
    except (InvalidOperation, TypeError) as exc:
        raise ValueError("amounts must be decimal-compatible") from exc

    if subtotal + tax != total:
        raise ValueError("subtotal plus tax must equal total")
    return value


class ExtractionRepository:
    def __init__(self) -> None:
        self._rows: dict[tuple[str, str], AcceptedExtraction] = {}

    def commit(self, item: AcceptedExtraction) -> AcceptedExtraction:
        key = (item.tenant_id, item.request_key)
        existing = self._rows.get(key)
        if existing is not None:
            return existing
        self._rows[key] = item
        return item
```

This validator intentionally rejects extra fields. That is a trade-off: it catches silent contract drift, but a schema addition now requires an explicit version transition. For an accounting path, surprise compatibility is worse than a controlled migration. The arithmetic rule is also intentionally narrow; discounts, withholding, rounding, and multiple tax rates require a richer schema and policy rather than another permissive cast.

Prompt instructions can request JSON, but instructions are not enforcement. Prompt changes should therefore carry versions, run against a fixed invoice corpus, and be evaluated on parsed fields rather than on prose similarity. Embeddings may help retrieve supplier-specific terminology or select related examples, yet they do not validate totals and should not be treated as proof that an extracted field is correct.

## A neutral acceptance test beats a feature matrix

Do not begin with a product checklist. Give every candidate the same adapter contract and the same tests. The candidate passes only if the service can authenticate before work begins, propagate cancellation, bound buffering, distinguish retryable transport failure from invalid output, and return a terminal object that survives the exact validator used in production.

The correctness corpus should contain clean invoices and hostile shapes: repeated invoice numbers across different tenants, missing currency, comma and period variations, negative adjustments, very long line-item tables, scanned pages with weak text, prompt-like text printed inside the invoice, and a connection dropped immediately before the terminal event. Label expected fields independently of the candidate response. Report exact field acceptance, rejection reason, duplicate-commit count, incomplete-stream count, and bytes retained per attempt. A single aggregate score hides the failures that matter.

Use a two-stage gate. First, require zero unauthorized cross-tenant reads and zero duplicate commits in the test run; these are invariants, not weighted preferences. Second, compare valid-result rate, error classification, tail latency, cancellation behavior, operational effort, and retained bytes under the workload you expect. Cost belongs in this second stage as measured resource use and quoted commercial terms at evaluation time, not as a permanent claim in an architecture note.

| Decision axis | Test | Reject when |
|---|---|---|
| Authorization | Request another tenant's operation identifier | Any document, fragment, or result is disclosed |
| Structured correctness | Parse and validate every terminal candidate | Invalid data is marked accepted |
| Replay safety | Repeat one idempotency key during and after a disconnect | More than one accepted record exists |
| Stream behavior | Slow, cancel, and reconnect the consumer | Memory is unbounded or terminal state is ambiguous |
| Retention control | Trace one request through logs and stores | Content survives outside the declared policy |
| Portability | Replace the adapter in the harness | Browser protocol or stored schema must change |

Products can differ in response schemas, streaming semantics, structured-output controls, quotas, and cancellation behavior, and those boundaries can change. Verify them during the evaluation against current primary documentation and the harness rather than copying a comparison table that will decay. The durable choice is the contract you own and the evidence you measure.

## What do we deliberately stop keeping?

Stop keeping routine fragment bodies, accumulated partial responses, unrestricted request and response logs, and duplicate normalized documents whose only purpose was debugging. Also stop treating chat history as an audit record. Retain the authenticated operation identifier, tenant scope, source-document reference, content hash where appropriate, schema and prompt versions, accepted JSON, validation status, usage counters, and timestamps according to explicit policies.

There is a real price. When an intermittent ordering defect appears outside the diagnostic sample, you may know that a stream was incomplete but lack the exact fragments needed to reconstruct its visual corruption. When a prompt regression is discovered after source evidence expires, you cannot replay that document. Short retention can also make a later supplier dispute harder to investigate. Those losses are why expiry should follow legal and operational requirements, diagnostic sampling should be deliberate, and deletion should be tested rather than assumed.

Keep less, knowingly.

The final selection rule is plain: choose the backend candidate that passes tenant isolation and idempotency invariants, produces the highest rate of schema-valid and domain-valid invoice objects on your fixed corpus, and can stream through your small protocol without forcing provider-specific state into the browser or durable store. Everything else is revisitable. The stored evidence model is not.

## Further reading

- OpenAI, "Embeddings guide": https://platform.openai.com/docs/guides/embeddings
- Prompt Engineering Guide: https://www.promptingguide.ai
