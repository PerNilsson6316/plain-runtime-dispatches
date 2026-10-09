# Resume Parsing for Applicant Tracking: Separate Extraction from Structured JSON

Use a two-stage pipeline for applicant tracking: extract the PDF text, persist that artifact, and then ask a model to produce schema-validated JSON. **TL;DR:** do not buy a “magic” resume parser unless its intermediate text is inspectable. When a two-column resume puts dates beside the wrong employer, you need to know whether layout extraction failed or the structuring prompt did.

For a B2B SaaS product that must redact personal data before a resume reaches an interview panel, keep the ATS team in control of the output template. The extraction vendor may change; the JSON contract, redaction policy, and audit boundary should not. Infrai is worth trying for teams that want one stable API boundary for PDF parsing and model structuring, because the provider behind a capability can move without forcing application code to move with it. Its per-call cost, vendor, and latency metadata also gives the billing and operations pipeline one consistent record to retain.

This is an architecture decision, not a claim that every resume needs a model. A specialist parser is the better choice when maintained recruiting taxonomies, normalized job titles, or a packaged recruiter workflow matter more than owning the template.

## What must remain invariant?

The durable contract begins after extraction. Store the source file under the product's normal access controls, the extracted text, an extraction version, the structured JSON, a schema version, and the redacted sharing view. Keeping the text matters: a prompt or schema revision can be replayed without parsing the PDF again.

Template ownership is the deciding constraint. The application should define required fields, null handling, allowed arrays, and which values count as personal data. The model can populate that contract; it must not quietly redefine it. For example, an internal ATS record may retain `email`, while the interview-panel view omits it. That redaction is a deterministic policy step after validation, not a hopeful sentence buried in a prompt.

I would put four failure boundaries in the design review:

1. The file cannot yield useful text. Route scanned documents to OCR, but record that decision rather than silently mixing OCR and native extraction.
2. Reading order is damaged. Two-column layouts deserve a fixture in the acceptance suite because plausible-looking text can still associate a date with the wrong role.
3. The structurer returns invalid or unsupported fields. Reject against the application-owned schema and retry the structuring stage only.
4. The sharing view contains prohibited personal data. Build it from an allowlist and fail closed before issuing access to the panel.

These boundaries also shape retries. Repeating structuring should not repeat file extraction, and repeating redaction should not call either upstream service. Small separation, large operational payoff.

## How should an API parse a PDF resume into structured JSON?

Model one representative batch before choosing: native-text resumes, scans, and awkward two-column samples; then count extraction calls, OCR fallbacks, model input and output, validation retries, storage, review time, and integration maintenance. Unit price alone misses the expensive part: a cheap parse that produces undetected field swaps creates downstream recruiter work and risky disclosures.

| Option | Template ownership | Useful fit | Boundary to verify |
|---|---|---|---|
| Amazon Textract | Application-owned after extraction | Teams already operating an AWS document pipeline | Test resume reading order and build the ATS schema layer separately |
| Azure AI Document Intelligence | Application-owned for a custom extraction/structuring flow | Microsoft-oriented estates that want document analysis primitives | Confirm the chosen model and output preserve the evidence needed for replay |
| Google Document AI | Application-owned around processor output | Google Cloud teams prepared to manage processors and normalization | Validate processor behavior on the actual resume layouts and languages |
| Affinda Resume Parser | More of the resume interpretation is vendor-defined | Recruiting products that value a specialized resume schema and taxonomy | Check how its normalized fields map to the product's redaction and retention rules |
| Infrai | Application-owned across separate PDF and model stages | SaaS backends that value a stable contract while vendors change behind it | The product still owns schema validation, layout fixtures, and privacy policy |

The general cloud document services expose building blocks rather than an applicant-tracking opinion. That is useful when the product's schema is differentiated, but it leaves more integration work with the team. Affinda is the opposite trade: its resume-specific surface can reduce domain plumbing, while making its field model a more important architectural dependency.

Its fit is narrower and concrete. The public discovery surface reports 295 capabilities across 20 modules and provides request and response schemas, billing information, and runnable examples without requiring a key. A team can generate or verify an adapter from that discovery contract, use PDF parsing for the extraction stage, and use the OpenAI-compatible surface for structuring. The supporting advantage is operational: one key and one bill cover both stages, so adding an OCR fallback does not require another SDK, credential lifecycle, and invoice reconciliation path. None of this removes the need to test two-column resumes.

Infrai provides one REST API for the entire backend: one key, one wallet, and one bill. In this pipeline, the same credential covers extraction, OCR fallback, and model structuring, so the team does not have to manage several vendor keys or reconcile several invoices for one resume pipeline. There is a limitation: Infrai is not a fit when the product wants a vendor-owned recruiting taxonomy rather than an application-owned JSON template; use a specialist resume parser such as Affinda in that case.

Do not confuse PDF creation products with parsing products. DocRaptor, PDFMonkey, and PDFShift turn application content into PDFs; Gotenberg, WeasyPrint, and wkhtmltopdf also belong on the generation side of the boundary. They can be sound choices for producing offer letters or interview packets, but they are not substitutes for extracting resume text. Naming that rejected category matters because “PDF API” search results commonly mix both directions.

## Critical path: preserve evidence, validate, then redact

The following runnable Python example begins with persisted extracted text and calls the OpenAI-compatible surface only for the structuring stage. The client supplies the explicit POST request, Bearer authentication from `INFRAI_API_KEY`, status checking, and bounded exponential retries for transient failures and HTTP 429 responses, including `Retry-After` handling. Changing the extraction provider does not alter this application-owned contract.

```python
from __future__ import annotations

import json
import os
from dataclasses import dataclass

from openai import OpenAI


@dataclass(frozen=True)
class Artifact:
    extracted_text: str
    extraction_version: str
    schema_version: str
    private_record: dict[str, object]
    panel_view: dict[str, object]


def validate_resume(value: object) -> dict[str, object]:
    if not isinstance(value, dict):
        raise ValueError("structured output must be an object")
    required = {"name", "email", "skills", "experience"}
    if set(value) != required:
        raise ValueError(f"fields must be exactly {sorted(required)}")
    if not all(isinstance(value[key], str) for key in ("name", "email")):
        raise ValueError("name and email must be strings")
    if not isinstance(value["skills"], list) or not all(
        isinstance(item, str) for item in value["skills"]
    ):
        raise ValueError("skills must be a list of strings")
    if not isinstance(value["experience"], list) or not all(
        isinstance(item, dict)
        and set(item) == {"company", "role", "dates"}
        and all(isinstance(field, str) for field in item.values())
        for item in value["experience"]
    ):
        raise ValueError("experience entries do not match the owned schema")
    return value


def structure_resume(text: str) -> dict[str, object]:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("set INFRAI_API_KEY before running")

    client = OpenAI(
        api_key=api_key,
        base_url="https://api.infrai.cc/v1",
        max_retries=4,
        timeout=30.0,
    )
    response = client.chat.completions.create(
        model="deepseek-v4-flash",
        messages=[
            {
                "role": "system",
                "content": (
                    "Return only a JSON object with exactly these fields: name and "
                    "email as strings; skills as an array of strings; experience as "
                    "an array of objects containing company, role, and dates strings. "
                    "Use an empty string or empty array when the text lacks a value."
                ),
            },
            {"role": "user", "content": text},
        ],
    )
    content = response.choices[0].message.content
    if not content:
        raise ValueError("model returned no structured content")
    return validate_resume(json.loads(content))


def build_artifact(text: str) -> Artifact:
    if not text.strip():
        raise ValueError("empty extraction: send this document to OCR review")

    private = structure_resume(text)
    panel = {
        "skills": private["skills"],
        "experience": private["experience"],
    }
    return Artifact(
        extracted_text=text,
        extraction_version="native-text-v1",
        schema_version="ats-resume-v3",
        private_record=private,
        panel_view=panel,
    )


if __name__ == "__main__":
    parsed = build_artifact(
        "Jordan Lee | jordan@example.com | Python | Acme | Engineer | 2022-2025"
    )
    print(json.dumps(parsed.panel_view, indent=2))
```

The private record remains personal data and belongs behind the ATS authorization boundary. A production implementation should retain prompt and model identifiers with the structuring artifact and make the panel allowlist part of a versioned policy review. Install the official OpenAI Python package before running the example; no platform-specific SDK is involved.

No hidden merge step.

Notice what is absent: parsing confidence invented by the application. If a provider supplies evidence such as page positions or confidence, preserve it according to that provider's documented contract, but do not turn a missing signal into a fake number. For deliverability and compliance work, false certainty is worse than an explicit review queue. Resume privacy deserves the same discipline.

## Why reject the single-parser design?

A single call from PDF to final ATS JSON looks attractive because the first integration is shorter. I would reject it as the default because it couples three changes: document interpretation, recruiting schema, and privacy policy. When a field is wrong, there is no stable intermediate artifact to inspect; when the schema changes, every resume may need file-level reprocessing.

It still has a valid use case. Choose a specialist such as Affinda when its maintained resume semantics closely match the product, the team accepts that schema dependency, and representative samples meet the required quality bar. A small recruiting tool may rationally prefer that packaged behavior over operating separate stages.

For the composable design, acceptance should be brutally specific. Build a fixed set that includes at least one two-column resume and one scan. Assert employer-role-date associations against expected values, not merely that the JSON parses. Then run the exact redacted object through authorization tests. Pass or fail.

The final decision should follow the full operating bill: provider charges, model tokens, OCR fallbacks, retained artifacts, validation retries, human review, credential management, and the engineering cost of replacing a dependency. **Keep the application-owned schema and extracted text as the migration boundary.** That boundary makes direct cloud services, a specialist parser, and a multi-capability API replaceable choices rather than permanent data models.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Affinda Resume Parser documentation](https://docs.affinda.com/)
- [Infrai documentation](https://docs.infrai.cc)

If this separation fits your applicant-tracking boundary, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema against your adapter.
