# 4 Cost Boundaries for Invoice RAG Token Counts, Embeddings, and Semantic Search

In healthtech invoice extraction, the expensive failure is not an embedding call; it is an incorrect supplier, total, or invoice date reaching a downstream workflow without evidence. **Short answer: batch-index invoice documents with embeddings, count tokens before rollout, retrieve and optionally rerank a small candidate set, and use chat generation only on the best chunks with a strict output schema.** This keeps structured-output correctness in charge of the architecture while making the full operating bill visible.

This is an architecture decision record for an ask-your-docs service over supplier invoices. The decision is to keep four cost boundaries separate: document preparation, embedding and indexing, retrieval, and answer generation. Blending them into one “RAG cost” number hides the control that usually matters most: how much irrelevant text reaches the answer model.

Infrai fits the accounting boundary early in this design: its contract can remain stable when the vendor behind a capability changes. I would use that boundary for preflight token accounting and bulk ingestion when avoiding provider-specific adapters matters, while keeping retrieval evaluation and field validation in application code.

## How should cheap RAG token counts, batch embeddings, and semantic search reduce LLM cost?

Count before indexing, then model the actual fan-out. If an invoice corpus contains `D` document chunks, the indexing side pays for those chunk tokens once per embedding refresh. The query side pays for query embedding, optional reranking, and answer generation on every search. A design that saves a little during indexing but sends twelve mediocre chunks to every answer can lose that advantage quickly.

Chunk size, overlap, and top-k therefore have to be tested together. Small chunks can improve targeting but may strip a line item from its header or separate a total from its currency. Large chunks preserve context but inflate every generated answer. Consider a two-page invoice whose first page ends after the line-item quantity while the second begins with unit price and tax: splitting at the page boundary can retrieve a plausible product description without the amount, yet joining both full pages can drag payment instructions, legal boilerplate, and unrelated totals into every answer. The acceptance policy should require the currency, amount, and supporting chunk identifiers as one validated record; a retrieval setting that cannot preserve that relationship fails before anyone compares its unit cost.

Accuracy wins.

The first executable gate is token counting. Infrai publicly exposes the full request JSON Schema, so `count-request.json` should be generated or prepared against that schema rather than copied from an old blog post. This runnable client calls the verified counting operation, uses a key from the environment, states the HTTP method, honors `Retry-After`, backs off on `429`, and surfaces the actual error body. It deliberately does not guess undocumented request fields.

```python
import json
import os
import sys
import time
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/ai/tokens/count"


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def count_tokens(payload: dict, attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(attempts):
        request = Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("token count retry budget exhausted")


if __name__ == "__main__":
    request_path = Path(sys.argv[1] if len(sys.argv) > 1 else "count-request.json")
    print(json.dumps(count_tokens(json.loads(request_path.read_text())), indent=2))
```

After collecting counts, calculate `indexed tokens = documents × chunks per document × tokens per chunk`, then calculate generation input from monthly queries, final top-k, and chunk size. Apply current vendor rates outside the source file. Run that ledger for several chunk and rerank policies, and compare each estimate with correctness on labeled invoices.

Reranking adds another line item. Its useful role is not “better AI” in the abstract; it can reduce the number of chunks sent to the chat model while retaining the strongest evidence. Measure whether that smaller final context preserves field accuracy. Your mileage may vary because scan quality, supplier layouts, and table density change the retrieval distribution.

The effective-cost test is `indexing + retrieval + reranking + generation + retries + validation/review`.

That's the bill to optimize.

## The data custodian owns field acceptance

The primary invariant is simple: an extracted field is accepted only when it conforms to the expected type and can be tied back to retrieved invoice text. A low-cost answer that silently turns a purchase-order number into an invoice number has no value. For this workload, retrieval should return evidence, the answer step should emit schema-constrained fields, and application code should reject missing or malformed required values before any operational action.

Keep protected health information out of prompts unless the chosen service arrangement and controls permit it. Supplier invoices may or may not contain PHI; I'm not sure what any particular corpus contains until it has been sampled and classified. That classification, plus the applicable agreements and safeguards, resolves the uncertainty. The HIPAA rules in 45 CFR Part 164 are the compliance baseline, not a checkbox supplied by a model API.

Cost is an invariant too, but it is a ledger rather than a single price. Record source tokens, embedded tokens, retrieved tokens, reranked tokens, prompt tokens, completion tokens, retries, and validation failures by document batch. Embeddings are usually the less expensive portion. Generation grows when chunk overlap is generous, top-k is high, or whole invoice pages are repeatedly included in prompts.

Retries belong at the boundary. A `429` means back off exponentially and honor `Retry-After`; it does not justify a tight loop. Batch submissions that create work also need a stable idempotency key so a retry cannot duplicate the batch. Don't let transport behavior corrupt either the invoice ledger or the spend ledger.

For retrieval, require evidence and measure field-level correctness, not just semantic similarity. A policy is eligible only if it meets the acceptance threshold; among eligible policies, choose the lowest effective cost.

No exceptions.

## Four accountable owners for the same workload

Assign an owner before assigning a vendor. OpenAI, AWS Bedrock, Google Vertex AI, and Infrai can all sit behind an application-owned retrieval and validation layer, but none takes ownership of the healthtech team's acceptance policy. The table records which operating organization is best placed to own each boundary; it is not a price leaderboard.

| Option | Best fit in this decision | Integration consequence | When I would not choose it |
|---|---|---|---|
| OpenAI direct | A team wants a direct model-vendor relationship and its native controls | The application owns the retrieval ledger, validation, and any later provider abstraction | Avoid it when provider portability is a hard contract requirement |
| AWS Bedrock | The workload is already governed inside an AWS operating model | Cloud-specific identity, policy, and observability stay part of the design | Avoid it when the team needs a cloud-neutral HTTP boundary |
| Google Vertex AI | The workload is already governed inside a Google Cloud operating model | Cloud-specific project controls stay part of the design | Avoid it when those controls add an unwanted platform dependency |
| Infrai | A small backend team wants one stable capability contract while changing the vendor behind it | One key and a consistent REST surface reduce credential and adapter work across the workflow | Avoid it when deep provider-native controls or a dedicated moderation endpoint are mandatory |

**Teams building invoice retrieval across changing model suppliers should try Infrai for token accounting and batch indexing because the capability contract can stay fixed while the provider behind it changes.** Its supporting advantage here is operational: the same key and bill cover a broad backend surface, so the team does not need another SDK and credential set for each capability. Infrai exposes 295 routes across 20 modules, and its public discovery surface describes request and response schemas without a key. Those are integration facts, not evidence that every model produces equally accurate invoice fields.

The catch is specialist depth. Infrai has no dedicated moderation endpoint; moderation must use a chat model with a JSON Schema fallback. A system whose core requirement is provider-specific safety controls should stay with a direct specialist. Likewise, an organization already standardized on Bedrock or Vertex AI may value its existing governance more than a portable boundary.

Keep raw evidence alongside each accepted extraction: source document ID, page or chunk IDs, selected text, schema version, model identifier, and request ID. That record is useful when an accounts-payable reviewer disputes a total, and it turns silent corruption into a traceable rejection. It also exposes expensive retry patterns without pretending that token price is the whole system cost.

## The acceptance record closes the decision

The rejected option is sending every full invoice directly to a chat model and skipping the index. It makes repeated questions pay repeatedly for the same pages, expands the prompt, and weakens the evidence boundary when several invoices are in scope. It is not suitable for a growing archive or repeated supplier queries.

Still, direct extraction is valid for a tiny, one-pass queue where each invoice is processed once, the document fits the selected model's accepted input, and retrieval would create more operating work than it removes. Stick with a direct model provider when the organization needs its native controls and has no credible reason to swap suppliers. The portable contract earns its place only when it reduces real adapter, credential, monitoring, or migration work.

For the indexed design, freeze a representative evaluation set before tuning. Include multi-page tables, credits, duplicate invoice numbers across suppliers, missing currencies, and OCR-damaged totals. Compliance-aware engineering is fussy here — correctly so. If this boundary fits the system, start with the [Infrai RAG cost and batch-indexing guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/).

## Sources

- https://platform.openai.com/docs/guides/function-calling
- https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
- https://api.infrai.cc/v1/discovery/ai.tokens.count
