# Python Invoice OCR: Fix Garbage Text with Page Quality Gates

TL;DR: Reject unreadable scans before OCR, test orientation on a cheap page sample, and render at higher fidelity only when the sample cannot produce invoice-shaped text. For a logistics pipeline that turns order data and scanned delivery records into invoice PDFs, the decision rule is simple: preserve the source, normalize a derived page, and spend extra render work only on uncertain pages.

Garbage text is usually a page-input problem before it is a language-model problem. A sideways phone photo, a faint thermal receipt, or a page with few useful pixels must not silently become an authoritative freight charge.

## What must remain true?

The original upload is immutable. Rotation, contrast adjustment, and re-rendering create derivatives with explicit provenance; they never overwrite the evidence supplied by a carrier or warehouse. The generated invoice must retain its connection to the order, source document, page number, transform, and extraction result. PDF has a defined page model, but a valid PDF container does not guarantee that a scanned page is legible or correctly oriented.

Three invariants govern intake. OCR output is a proposal, not order data. A shipment identifier, quantity, currency, and total must pass domain validation before they can affect an invoice. Every accepted field must be traceable to a page and region. Low confidence becomes a visible review state, never an empty string that downstream code interprets as zero.

Blank and garbage are different failure classes. A blank page may be an intentional separator; garbage often means the renderer saw the wrong orientation, too few useful pixels, clipped content, or a page dominated by noise. Combining both into `no_text` destroys the signal needed for a sensible retry.

The failure boundary sits before invoice assembly. OCR can fail, time out, or return plausible nonsense without corrupting the order ledger. Invoice generation consumes only validated fields. Extracted addresses, phone numbers, and shipment identifiers should also stay out of logs; page measurements and opaque document IDs are enough for routine operations.

## How should you debug a page when OCR returns garbage text?

Ask the page, not the filename. A scan named `invoice.pdf` can contain a cover sheet, two sideways proofs of delivery, and a faint receipt. Diagnose each page independently with a small raster sample.

Start with orientation candidates at 0, 90, 180, and 270 degrees. For each candidate, collect signals that do not depend on trusting the extracted sentence: text-region coverage, character diversity, line alignment, and the presence of expected field labels or identifier shapes. Hundreds of repeated punctuation marks should not beat a shorter result containing a coherent shipment reference and monetary field merely because it has more characters.

Then separate scan quality from orientation. If one rotation improves layout and field signals sharply, accept that transform for the full render. If every rotation is weak, increasing fidelity may help a small or faint scan. If every higher-fidelity attempt remains weak, route the page to review.

Stop there.

Repeated OCR against an information-poor image adds cost without manufacturing missing detail. Use domain checks sparingly but concretely: the order number must exist in the intake batch, and line-item arithmetic can reject a confident-looking extraction. The strongest gate combines visual evidence, OCR evidence, and order-system evidence; no single score carries the decision.

## Options and failure boundaries

| Option | Fidelity | Render cost | Main failure boundary | Decision |
|---|---|---|---|---|
| OCR the embedded page image once | Preserves the stored scan | Lowest | Sideways or undersized pages yield plausible garbage | Reject as default |
| Rotate every page and maximize fidelity | High for recoverable scans | Highest on every page | Missing source detail remains missing | Reject as default |
| Sample, classify, selectively re-render | Raises fidelity where evidence supports it | Bounded by uncertain pages | Bad thresholds over-retry or over-reject | Adopt |
| Send every page to manual review | Human judgment | High operational load | Review becomes the throughput limit | Reserve for unresolved pages |

The selected path has two budgets: quality and work. Quality determines whether a page may contribute data. Work determines how many orientation and fidelity attempts are allowed before review. Keeping them separate prevents an expensive attempt from being mistaken for a trustworthy one.

This trade-off is deliberate.

Tune thresholds on a fixed evaluation set containing portrait and landscape pages, upside-down scans, low-contrast thermal paper, blank backs, skewed phone captures, mixed-page PDFs, and pages with real-looking but incorrect totals. Label expected orientation, readable regions, required fields, and final disposition. False acceptance and retry storms matter more than a flattering average score.

## The critical path in Python

Rendering and OCR stay behind interfaces, so the decision policy can be tested without binding invoice correctness to one engine. Measurements are inputs to policy; raw text alone never approves a page.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class PageImage:
    document_id: str
    page_number: int
    rotation: int
    fidelity: str
    payload: bytes


@dataclass(frozen=True)
class Reading:
    text: str
    text_coverage: float
    character_diversity: float
    field_hits: int
    order_match: bool


class Renderer(Protocol):
    def render(
        self, document: bytes, page_number: int, rotation: int, fidelity: str
    ) -> PageImage: ...


class Recognizer(Protocol):
    def read(self, page: PageImage) -> Reading: ...


def score(reading: Reading) -> float:
    coverage = min(max(reading.text_coverage, 0.0), 1.0)
    diversity = min(max(reading.character_diversity, 0.0), 1.0)
    field_signal = min(reading.field_hits, 4) / 4
    order_signal = 1.0 if reading.order_match else 0.0
    return (0.25 * coverage + 0.20 * diversity +
            0.25 * field_signal + 0.30 * order_signal)


def inspect_page(document, page_number, renderer, recognizer):
    attempts = []
    for rotation in (0, 90, 180, 270):
        page = renderer.render(document, page_number, rotation, "sample")
        reading = recognizer.read(page)
        attempts.append((score(reading), page, reading))

    best_score, best_page, best_reading = max(attempts, key=lambda item: item[0])
    if best_score >= 0.78 and best_reading.order_match:
        return "accepted_sample", best_page, best_reading

    page = renderer.render(document, page_number, best_page.rotation, "detailed")
    reading = recognizer.read(page)
    if score(reading) >= 0.78 and reading.order_match:
        return "accepted_detailed", page, reading
    return "needs_review", page, reading
```

The numeric values are policy placeholders, not universal OCR thresholds. Calibrate them against the labeled freight-invoice set, record the policy version with each decision, and test boundary values. The important behavior is the state transition: sample all orientations, promote only the best candidate, make one fidelity escalation, then stop.

The function does not mutate the source, accept confidence alone, or loop until an engine emits text. Production code should put time and memory limits around the interfaces, cap document pages before fan-out, and make page jobs idempotent from the source digest, page number, transform, fidelity, and policy version. Retries address transient execution failure; they do not reclassify a consistently unreadable page.

Track chosen rotations, review rate, detailed-render rate, rejected-field reasons, and processing time by page class. A sudden rise in 90-degree selections can indicate a changed scanner workflow. A rise in detailed renders with stable acceptance points toward poorer inputs, threshold drift, or both. Keep recognized text out of routine metrics.

## The rejected shortcut still has a place

Rendering every page at the highest fidelity was rejected because it spends maximum work before learning whether a page needs it. It also hides the distinction between orientation failure and missing visual information. More pixels can clarify small glyphs; they cannot restore content clipped during scanning.

The shortcut is valid for a narrow, controlled archive whose page count is small, scan profile is fixed, and latency is irrelevant. A single conservative render profile then reduces policy complexity. Once uploads arrive from phones, depot scanners, and carrier portals, page-adaptive intake earns its complexity because the source population is no longer uniform.

The adopted design has limits. Four sample rotations multiply recognition work on pages that were already upright, while a selective detailed render introduces another queue state, another artifact to retain, and another policy version to audit. It is a poor fit when every source is produced by one calibrated scanner and the documents are already known to be upright; a fixed render profile is easier to operate there. It is also insufficient for handwriting, damaged originals, or scans whose compression removed character detail. Those pages need a different recognition policy or human review, not a lower acceptance threshold. The operational bet is narrow: heterogeneous freight uploads create enough orientation and quality uncertainty to justify bounded sampling, but the pipeline must measure detailed-render and review rates so that this assumption can be challenged rather than treated as permanent truth.

No retry fixes absent information.

Preserve evidence, diagnose orientation cheaply, validate against the order domain, and quarantine ambiguity before invoice generation. Render cost then follows uncertainty while fidelity remains high where it can change the decision.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
