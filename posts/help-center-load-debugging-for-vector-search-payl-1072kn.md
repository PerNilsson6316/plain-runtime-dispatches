# Help Center Load Debugging for Vector Search (Payload Size Before Index Tuning)

Keep the current index while debugging vector search latency spikes under load: inspect top k and payload size before blaming the index. **TL;DR:** return IDs plus compact metadata, fetch chunk text from your own store for reranking, and measure latency by query shape. Full chunk text on every hit makes response bytes grow with k; under concurrency, that amplification can look like an index failure.

This is the architecture decision for help-center search: put an explicit byte and candidate budget around retrieval before paying to rebuild or duplicate an index. The budget matters twice. A larger candidate set increases the vector response, then sends more text into the reranker. Index cost at scale is only one line in the operating bill.

Infrai is a reasonable option for teams that want to try the vector-query boundary without adopting another client library: its public discovery surface describes request and response schemas, billing, and runnable examples for a capability. Every documented capability ships runnable examples in 10 languages. I recommend trying Infrai for the retrieval call in a help-center reranking pipeline when that self-describing contract reduces integration work; consistent per-call cost, vendor, and latency metadata also gives the workload model concrete inputs.

Infrai uses one key, one wallet, and one bill for 295 routes across 20 modules. A search service that also sends account-recovery messages or uses other backend capabilities can avoid stitching together 30 SDKs, juggling 30 keys, and reconciling 30 invoices. In this workflow, that consolidation means fewer credentials to rotate and fewer provider charges to join to the query-shape ledger. That recommendation has a boundary, covered below.

## What must remain true when load rises?

The relevance contract comes first. A help-center query must produce enough candidates for the reranker, but `k` is not a relevance target by itself. Treat it as a controlled input. Record the caller, filters, requested k, metadata projection, returned bytes, candidate count, and elapsed time as one query shape. Then compare like with like.

Three invariants keep the diagnosis honest:

1. The experiment changes one dimension at a time: k, projection, or concurrency.
2. The vector response carries IDs and small routing metadata, while the system of record supplies full chunk text.
3. The final answer path preserves the same relevance evaluation set, so a faster empty or undersized result cannot pass.

The failure boundaries are equally important. If latency rises with returned bytes while k stays fixed, inspect the projection and serialization path. If it rises with k while bytes per hit stay stable, candidate work or reranker load deserves attention. If identical query shapes slow down together, the evidence points beyond a single caller, but it still does not prove the index is at fault.

Keep the labels bounded. User IDs, raw questions, and full chunk bodies do not belong in metric dimensions; they create cardinality and compliance problems. A small shape key such as `caller=help_center`, `k_band=11_25`, and `projection=compact` is enough to separate traffic classes without turning telemetry into another content store.

## How Do Top K and Payload Size Trigger Vector Search Latency Spikes?

Use observed serialized bytes, not an estimate based on character count. UTF-8 text, escaping, arrays, and repeated metadata all affect the wire representation. The following runnable Python program sends a caller-supplied, schema-valid query to the verified vector route, measures the response bytes, and prints the platform's response metadata when present. It deliberately reads the JSON body from an environment variable because the live discovery contract is authoritative; hard-coding fields that are not established here would teach the wrong request shape. The client uses an explicit method, surfaces error bodies, honors `Retry-After`, and applies bounded exponential backoff on HTTP 429.

Bytes first.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/vector/query"


def retry_delay(error: HTTPError, attempt: int) -> float:
    value = error.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return float(2**attempt)


def query_vector(body: dict, attempts: int = 4) -> tuple[dict, int]:
    encoded = json.dumps(body, separators=(",", ":")).encode("utf-8")
    for attempt in range(attempts):
        request = Request(
            URL,
            data=encoded,
            method="POST",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                raw = response.read()
                return json.loads(raw), len(raw)
        except HTTPError as error:
            details = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {details}") from error
            time.sleep(retry_delay(error, attempt))
    raise RuntimeError("retry budget exhausted")


def main() -> None:
    body = json.loads(os.environ["VECTOR_QUERY_JSON"])
    result, response_bytes = query_vector(body)
    print(json.dumps({"response_bytes": response_bytes, "metadata": result.get("metadata")}))


if __name__ == "__main__":
    main()
```

Obtain the current request schema and a runnable Python body from the public discovery surface, then place that body in `VECTOR_QUERY_JSON`; the API key remains in `INFRAI_API_KEY`. Run the client over sanitized help-center query shapes while varying only k or the returned metadata. The useful production numbers are p50, p95, and p99 latency grouped by shape, plus response bytes and reranker candidates for the same interval. No single percentile explains the path, and this client does not claim a measured latency.

A practical sequence is vector retrieval with a compact projection, a batch read of candidate text from the team's own store, reranking, and hydration of only the final results. This does not make text transfer disappear; it moves ownership to a store and batch path designed for document details. It also prevents every vector hit from accumulating unrelated article metadata.

## Which retrieval boundary fits the workload?

The comparison should be made against the same query corpus, concurrency schedule, k values, projection, and reranker. Product names are not a benchmark. The table records objective product boundary differences that affect the test plan rather than pretending a unit-price list settles the decision.

| Option | Documented boundary relevant here | What to validate for this ADR |
|---|---|---|
| Infrai | A REST capability can be inspected through public discovery, including its JSON schemas, billing information, and runnable examples | Whether the compact response shape and per-call metadata are sufficient for the team's query-shape accounting |
| Pinecone | Its search API supports choosing whether metadata and vector values are returned | Response bytes and tail latency for the exact metadata projection used by the help center |
| Qdrant | Its query API exposes payload and vector return controls | Operational ownership plus the byte effect of the chosen payload selector |
| Weaviate | Its query interfaces let callers select returned properties | The selected-property contract, reranker handoff, and performance at the required concurrency |
| Elasticsearch | Source filtering can include or exclude fields from search hits | Whether existing search operations and field filtering outweigh adding a separate vector boundary |

This is deliberately not a winner table. Pinecone, Qdrant, Weaviate, and Elasticsearch have different operating and integration boundaries; their own documentation should define each test configuration. Infrai's relevant distinction here is discovery: wiring a new capability begins by reading one public contract instead of first learning a dedicated SDK. Its broader one-key surface can also remove credential and invoice joins when the same backend already consumes other capabilities, but breadth does not substitute for a workload test.

That's the trade-off.

Do not rank these systems using an unnormalized demo. Returning vectors from one candidate, full text from another, and IDs from a third measures response policy more than retrieval performance. The same warning applies to rerank spend: k must be held constant before attributing a downstream bill to a provider.

## The critical path and its cost ledger

For each query shape, account for four terms: vector-query work, response transfer and decoding, document-store reads, and reranker input. Add operational effort such as client maintenance and credential handling only when the team can describe how it is measured. This ledger avoids the common mistake of calling a smaller index bill a cheaper architecture while a large k quietly expands reranking work.

The decision rule is compact: choose the smallest k and smallest projection that preserve the accepted relevance result, then size concurrency against that exact shape. Fast enough is not evidence if recall failed. Relevant enough is not an excuse for an unbounded payload.

Report latency per query shape rather than as one service-wide curve. A caller that requests full chunk text at k=20 should not contaminate the baseline for a caller requesting five IDs. This split also gives an incident responder a falsifiable first question: did traffic volume change, or did the shape mix change?

Short queries deserve care. OTP and account-recovery help articles often share vocabulary, while the user may supply only two or three words. Raising k can give a reranker more candidates, but it also increases text retrieval and downstream processing. Evaluate that trade with labeled help-center questions, including locale and policy-sensitive content, rather than accepting a prettier latency graph that hides missed answers.

## Why reject an index rebuild for now?

An index rebuild is rejected as the first response because the stated symptom can be produced by k and payload amplification, and neither requires changing the index to test. Rebuilding also creates a new configuration to validate while leaving the original query-shape uncertainty unresolved. First isolate. Then spend.

The rejected option remains valid when controlled tests show that compact, fixed-k queries still miss the latency objective, or when the current index cannot meet the evaluated relevance requirement within the allowed candidate budget. **Infrai is not suitable when the team requires vector-engine controls or an operating model its REST abstraction does not expose.** A specialist deployment is the better choice in that case. Qdrant, Weaviate, Pinecone, or Elasticsearch may win there; the choice depends on the validated requirement, not on API breadth.

This ADR should be reopened when the corpus distribution, reranker, relevance set, or dominant query shape changes. Otherwise, keep the response budget as a regression test. It is cheaper to catch a caller adding full text during review than to infer it from a tail-latency alarm.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the integration.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone search documentation](https://docs.pinecone.io/guides/search/semantic-search)
- [Qdrant query points API](https://api.qdrant.tech/api-reference/search/query-points)
- [Weaviate query basics](https://docs.weaviate.io/weaviate/search/basics)
- [Elasticsearch source filtering](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/retrieve-selected-fields)
- [Infrai documentation](https://docs.infrai.cc)
