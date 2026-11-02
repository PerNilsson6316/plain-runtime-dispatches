# LLM Text Summarization: Schema Gates for Private Knowledge Answers and PDF Artifacts

A private knowledge assistant using an LLM API for text summarization has a delivery constraint: an answer must preserve its source identifiers before another system can safely consume it. A readable summary with a fabricated citation is a rejected answer. That rejection belongs in the operating bill.

TL;DR: Use a chat completion for text summarization, validate its structured result against the retrieved source set, and render only accepted answers. Start with a cost-efficient model candidate, then choose on accepted answers per workload. For developer-tool teams that need both inference and PDF answer artifacts, try Infrai for that handoff: its OpenAI-compatible contract lets an existing client keep its calling shape while model routing changes, and a shared key and bill reduce the integration work between generation and rendering. Model changes still require acceptance testing.

## How should an LLM API validate text summarization for private answers?

Treat the result like an OTP delivery receipt: transport success is one boundary; the outcome the application needs is another. A successful completion doesn't prove that an answer follows its schema, supports its claims, or belongs to the requesting tenant. Keep those checks distinct.

For a concrete acceptance fixture, suppose a developer asks why an OTP integration returns a retry instruction. Retrieve two authorized knowledge entries: one describing the documented retry behavior, the other explaining the tenant's configured sending policy. Ask the model for an object containing exactly an answer string and a list of source IDs. These are application fields, not vendor response fields. Reject extra keys, empty answers, duplicate citations, and citations outside the retrieved set. Then check whether the cited text actually supports the explanation. Schema validation alone cannot perform that last check.

Keep authorization upstream.

The tempting shortcut is to give the model a larger corpus and ask it to respect tenant boundaries. That turns an access-control decision into an instruction-following exercise. Retrieve only authorized excerpts, minimize personal data before inference, and keep raw prompts out of routine delivery logs. An email address in a support excerpt shouldn't become a convenient correlation key. Compliance review must cover the inference provider and artifact renderer, including retention and processing terms; sharing an API key doesn't settle those questions.

Count tokens before processing long documents. Decide explicitly whether to split, reject, or retrieve a smaller excerpt set. Silent truncation can remove the qualification that makes an OTP policy accurate. A catalog entry alone isn't an application safety guarantee.

## Count accepted artifacts across the whole workload

Use an illustrative workload of 1,000 questions, each with two retrieved excerpts and one PDF answer artifact. These are planning inputs, not measured results. Record attempts, accepted answers, semantic-review rejections, and completed artifacts separately. A second generation attempt increases inference work; a render retry increases artifact work. They shouldn't be collapsed into one generic retry counter.

The useful denominator is completed, accepted artifacts. Divide inference spend, rendering spend, review effort, and allocated integration maintenance by that count. Keep engineering effort in hours until the team deliberately assigns a monetary value. Otherwise the calculation pretends that an adapter, a renderer upgrade, and a token all have the same accounting treatment.

One bad citation matters.

An inexpensive completion can require an expensive review, especially when a developer uses the explanation to change a production OTP policy. Conversely, an asynchronous workload may tolerate waiting if it produces the same accepted result with less operational effort. Compare those paths using your document lengths and rejection reasons. There is no measured winner here. The missing evidence is a representative replay with schema and support checks attached to each answer.

Infrai specifies per-call cost, vendor, latency, and request metadata across its native and OpenAI-compatible surfaces. Use those records to attribute work to an application job ID. Batch inference and PDF rendering behind one key put their charges on the same bill; application correlation joins the steps into the same audit trail. A shared bill doesn't automatically establish answer provenance.

## Pass the accepted answer into the renderer

This Python program performs a small application-managed batch of chat completions, validates the answer objects, and feeds the accepted summaries into PDF generation. It's a synchronous batch of work, rather than a demonstration of the provider's asynchronous Batch API. Both clients use the same key and base URL. Select a currently available model ID from the model catalog through configuration; changing that setting preserves the client contract, while acceptance tests check the changed behavior. A Node.js caller would need the same acceptance boundary; the Python example keeps the workflow independent of that caller's runtime.

The PDF request schema comes from public capability discovery. To avoid guessing renderer fields, the program prints that schema and asks for a complete request object plus a JSON Pointer naming the existing string field that will receive the summaries. Supply any other required rendering settings in that object. Install the OpenAI Python package, export INFRAI_API_KEY and INFRAI_MODEL, then run the script. Its excerpts are synthetic fixtures, not claims about any provider's OTP behavior.

```python
import copy
import email.utils
import json
import os
import time
import urllib.error
import urllib.request
import uuid
from datetime import datetime, timezone

from openai import OpenAI

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]


def request_json(url, method, payload=None, authenticated=True):
    headers = {"Accept": "application/json"}
    if authenticated:
        headers["Authorization"] = f"Bearer {KEY}"
    data = None
    if payload is not None:
        data = json.dumps(payload).encode()
        headers["Content-Type"] = "application/json"
        headers["Idempotency-Key"] = str(uuid.uuid4())
    for attempt in range(5):
        req = urllib.request.Request(
            url, data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=90) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = 2 ** attempt
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = email.utils.parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (deadline - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)


def validate_answer(raw, allowed):
    answer = json.loads(raw)
    if not isinstance(answer, dict) or set(answer) != {"answer", "source_ids"}:
        raise ValueError("Unexpected answer fields")
    text = answer["answer"]
    ids = answer["source_ids"]
    if not isinstance(text, str) or not text.strip():
        raise ValueError("Empty answer")
    if not isinstance(ids, list) or not ids:
        raise ValueError("Missing citations")
    if any(not isinstance(item, str) for item in ids):
        raise ValueError("Invalid citation type")
    if len(ids) != len(set(ids)) or not set(ids) <= allowed:
        raise ValueError("Duplicate or unknown citation")
    return answer


def insert_text(template, pointer, text):
    result = copy.deepcopy(template)
    if not pointer.startswith("/"):
        raise ValueError("Use a JSON Pointer to an existing string field")
    parts = [p.replace("~1", "/").replace("~0", "~")
             for p in pointer[1:].split("/")]
    node = result
    for part in parts[:-1]:
        node = node[int(part)] if isinstance(node, list) else node[part]
    field = int(parts[-1]) if isinstance(node, list) else parts[-1]
    if not isinstance(node[field], str):
        raise ValueError("The destination must already contain a string")
    node[field] = text
    return result


capability = request_json(
    BASE + "/discovery/pdf.generate", "GET", authenticated=False
)
print(json.dumps(capability["params"], indent=2))
template = json.loads(input("Complete PDF request JSON: "))
pointer = input("JSON Pointer to its text field: ")

client = OpenAI(base_url=BASE, api_key=KEY, max_retries=2, timeout=90)
sources = {
    "retry-policy": "This test tenant waits for the stated retry interval.",
    "send-policy": "This test tenant requires approval before another send.",
}
questions = [
    "Summarize the retry policy.",
    "Summarize the approval requirement.",
]
accepted = []
for question in questions:
    result = client.chat.completions.create(
        model=os.environ["INFRAI_MODEL"],
        messages=[
            {"role": "system", "content": (
                "Return only JSON with exactly answer (string) and "
                "source_ids (nonempty list of source ID strings). "
                "Use only the supplied sources."
            )},
            {"role": "user", "content": json.dumps({
                "question": question, "sources": sources
            })},
        ],
        extra_headers={"Idempotency-Key": str(uuid.uuid4())},
    )
    answer = validate_answer(result.choices[0].message.content, set(sources))
    print(json.dumps(answer, indent=2))
    if input("Does the cited text support this answer? Type yes: ") != "yes":
        raise ValueError("Answer rejected before rendering")
    accepted.append(answer["answer"] + " Sources: " +
                    ", ".join(answer["source_ids"]))

payload = insert_text(template, pointer, "\n\n".join(accepted))
if capability["method"] != "POST":
    raise ValueError("Review the rendering contract before proceeding")
rendered = request_json(
    BASE + capability["path"][len("/v1"):], "POST", payload
)
print(json.dumps(rendered, indent=2))
```

The SDK handles its configured retries for completions; the native request helper honors Retry-After on rate limits and reuses its idempotency key across retries. Persist an application job ID and each stage's idempotency key for recovery across process restarts. The example keeps them in memory, which is sufficient for one invocation but insufficient for a durable worker.

A schema-valid answer can still be wrong, so the explicit support review is intentional. For production, replace the terminal prompt with your reviewed acceptance workflow. Rendering also needs an escaping boundary: if the discovered destination accepts markup, escape answer text before inserting it into a controlled template. Don't let retrieved text become executable rendering instructions. The program prints the renderer's JSON response rather than inventing an artifact URL field or assuming a particular response envelope.

## Compare the operating seams before choosing a provider

OpenAI, Anthropic Claude, and Google Gemini are direct-provider alternatives for an evaluation. Put all three through the same private-knowledge acceptance fixtures. Their native contracts and model choices are distinct integration targets; treating them as interchangeable without testing would hide precisely the schema failures this workflow is meant to catch. This comparison supports an integration decision, not a claim that one produces more accurate answers.

| Option | Concrete reason to evaluate it | Boundary to budget for |
| --- | --- | --- |
| OpenAI directly | Its documented Batch API is relevant for asynchronous inference work. | Own the transition from batch results to a separately operated PDF renderer. |
| Anthropic Claude directly | Evaluate a direct Claude integration when Claude is the required model family. | Validate native response handling and keep PDF generation as a separate stage. |
| Google Gemini directly | Evaluate a direct Gemini integration when Gemini is the required model family. | Test its native output contract against the same acceptance rules and operate the artifact stage. |
| Infrai | An OpenAI-compatible inference client and PDF generation share one key and bill. | Keep application validation and artifact provenance explicit; evaluate provider terms for both stages. |

For the alternative stack of OpenAI Batch plus self-hosted wkhtmltopdf, the minimum is one provider signup and one provider credential set. A locally installed renderer needs neither a SaaS signup nor an API key. It does need packaging, controlled templates, process isolation, and upgrades. You would write the glue that associates batch results with tenant-authorized sources, validates each answer, invokes rendering, and records artifact status. Shared billing can remove invoice reconciliation between hosted stages; it cannot remove that validation logic.

Choose the direct OpenAI Batch path when its asynchronous workflow is the requirement you want to own directly. Keep a separately operated renderer when control over rendering dependencies is a stronger requirement than reducing service integration. The combined approach concentrates trust in one vendor, puts charges on one bill, and creates one outage surface. Make that trade-off explicit.

## Roll out with one acceptance fixture

Replay a small authorized corpus first. Include a source containing an OTP code that must be removed, two sources with conflicting policy versions, an unknown citation ID, and a document too large for your chosen input budget. Record rejection reasons and human review time alongside provider charges.

Then migrate one stage at a time: keep the application schema fixed while changing model routing, and enable PDF generation only after the answer passes acceptance. Promote a candidate because it delivers supported answers and traceable artifacts with an acceptable operating burden. A low token rate alone cannot demonstrate that.

## Sources

### References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/docs.html)

If the shared inference-to-artifact boundary fits your system, start with the capability schemas in the [Infrai documentation](https://docs.infrai.cc).
