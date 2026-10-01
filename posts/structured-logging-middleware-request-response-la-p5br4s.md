# Structured Logging Middleware: Request Response Latency and Status Code for Notifications

Instrument Express middleware once for structured logging of each request and response, then keep provider delivery outcomes as separate correlated events. For a notification service, that split is the cleanest way to explain both failure and cost: the API completion event captures latency and status code for what the application accepted, while later delivery events say what happened after acceptance.

TL;DR: capture a stable operation ID, notification channel, message class, tenant or cost center, response status, and monotonic elapsed time in middleware. Never treat an HTTP success as proof of email, SMS, or OTP delivery. Join later outcomes by operation ID, then attribute spend to the attempt that caused it rather than to whichever request happened to be open when a callback arrived.

This is an architecture decision record for application logging. The decision is deliberately vendor-neutral. An Express service can implement the boundary with Pino, another JSON logger, or a thin internal adapter; shipping can target any system that preserves JSON fields. The schema and event timing matter more than the transport.

## How should Express middleware log structured request and response data?

The first invariant is identity. Every accepted notification operation needs one application-generated identifier that survives queueing, retries, and delivery-status updates. A transport request ID can still be recorded, but it is evidence about one hop, not the durable identity of the notification operation.

The second invariant is temporal honesty. Request latency ends when the API response finishes. Delivery latency ends when a terminal delivery outcome is observed. Combining those clocks produces an attractive number with no stable meaning, especially when an OTP is queued quickly but reaches its destination later.

Two clocks. Two claims.

The third invariant is bounded cardinality. Fields used for aggregation should come from controlled sets: `channel=email|sms`, `message_class=otp|transactional`, `outcome=accepted|rejected|delivered|failed|unknown`. Recipient addresses, message bodies, authorization values, and raw query strings do not belong in routine logs. They create compliance exposure and make aggregation noisy. Keep secrets out.

Finally, cost attribution must tolerate partial knowledge. The application may know the tenant and intended channel at acceptance time while the final delivery outcome is still unknown. Record what is known on each event and update analytical state by correlation; do not rewrite history in the log stream.

Unknown is valid.

These rules define the failure boundaries. Middleware can observe request start, response completion, uncaught application errors passed through the framework, and client disconnects exposed by the host interface. It cannot prove downstream delivery. A delivery worker can observe attempts, and a status consumer can observe later outcomes, but neither should retroactively change the meaning of the API event.

## Choose the attribution record, not merely the logger

A JSON logger solves serialization. It does not decide which event owns cost. That decision affects incident review, finance reconciliation, and rate-limit policy, so it belongs in the architecture rather than in an ad hoc log message.

The trade-off is explicit.

| Option | Cost owner | Failure interpretation | Main trade-off |
|---|---|---|---|
| Request-completion event only | Incoming API call | Non-2xx means request failure | Easy to deploy, but it cannot represent delayed delivery or retry cost |
| One mutable notification record | Latest operation state | Current state replaces earlier state | Convenient for dashboards, weak as an append-only audit trail |
| Correlated immutable events | Individual attempt, grouped by operation ID | Each boundary reports only what it observed | Requires a join, but preserves retry count, timing, and attribution |

Choose correlated immutable events. **Charge the attempt, group by the operation.** If an SMS attempt is rejected before dispatch, its event can carry the appropriate billable classification as known by the application. If another attempt follows, it gets another event. The aggregation layer can then calculate attempts and outcomes without guessing from a final status.

Do not put a floating-point currency value into middleware just because the request selected a channel. Pricing and contractual rules change outside the request lifecycle. A more durable event carries cost dimensions such as tenant, channel, route class, region class, and attempt number. A controlled downstream process can apply the applicable rate card and retain its version alongside the derived charge. This separates operational truth from commercial policy.

That distinction matters during delivery gaps. A spike in accepted OTP requests with few terminal outcomes is not the same incident as a spike in explicit failures, even if a single success-rate chart makes them look similar. Preserve `unknown` as a real analytical state until evidence resolves it.

Silence is not delivery.

## Critical path: emit exactly one completion event

The middleware below is framework-neutral Python that shows the mechanism expected at an Express-style boundary. The host adapter supplies a request object, invokes the next handler, and calls `finish` once when the response is complete. In a Node.js implementation, Pino would receive the same dictionary-shaped event as structured fields rather than a formatted sentence.

```python
import json
import time
from dataclasses import dataclass
from typing import Callable, Mapping


@dataclass(frozen=True)
class RequestContext:
    operation_id: str
    method: str
    route_template: str
    tenant_id: str
    channel: str
    message_class: str


def completion_middleware(
    request: RequestContext,
    next_handler: Callable[[Callable[[int], None]], None],
    write_log: Callable[[Mapping[str, object]], None],
) -> None:
    started_ns = time.monotonic_ns()
    completed = False

    def finish(status_code: int) -> None:
        nonlocal completed
        if completed:
            return
        completed = True

        elapsed_ms = (time.monotonic_ns() - started_ns) / 1_000_000
        event = {
            "event_name": "notification_api_completed",
            "operation_id": request.operation_id,
            "method": request.method,
            "route_template": request.route_template,
            "tenant_id": request.tenant_id,
            "channel": request.channel,
            "message_class": request.message_class,
            "status_code": status_code,
            "latency_ms": round(elapsed_ms, 3),
        }
        write_log(event)

    next_handler(finish)


def write_json(event: Mapping[str, object]) -> None:
    print(json.dumps(event, separators=(",", ":"), sort_keys=True))
```

There are two small details with large consequences. `time.monotonic_ns()` measures elapsed time without depending on wall-clock adjustments. The `completed` guard prevents two completion signals from creating duplicate cost evidence. The status code is captured at completion, after application code has had the opportunity to set it.

A production adapter also needs an explicit policy for abnormal termination. Emit a distinct event such as `notification_api_aborted` when the connection closes before normal completion, and keep `status_code` absent if no response status was actually observed. Fabricating `500` would turn missing evidence into a false server response.

Delivery attempts belong on another critical path:

```python
def delivery_attempt_event(
    operation_id: str,
    attempt_id: str,
    tenant_id: str,
    channel: str,
    attempt_number: int,
    outcome: str,
) -> dict[str, object]:
    return {
        "event_name": "notification_delivery_attempted",
        "operation_id": operation_id,
        "attempt_id": attempt_id,
        "tenant_id": tenant_id,
        "channel": channel,
        "attempt_number": attempt_number,
        "outcome": outcome,
    }
```

Keep the vocabulary narrow and document it. RFC 5424 defines standardized severity levels for syslog, but severity should not carry business outcome by itself. A delivery rejection may deserve an error severity operationally; `outcome=failed` is still the field an analyst should aggregate. Severity answers urgency. Outcome answers state.

## Shipping, metrics, and failure handling

Write JSON to a local process stream and let a separate collector handle batching, buffering, and transport. Request middleware should not wait for a remote log API before returning a notification response. Coupling those paths adds a new downstream dependency to every email, SMS, and OTP request, and a logging outage can then become an application outage.

Backpressure still needs a declared policy. The process can use bounded buffering and expose dropped-event counts, while the collector can retry transport independently. For high-value audit events, the system may choose a durable local handoff instead. That is a workload decision: stronger durability adds I/O and operational complexity. Document which event classes may be sampled or dropped. Completion and delivery-attempt events used for cost attribution should not be sampled casually.

Logs and metrics answer different questions. OpenTelemetry describes a metric as a measurement captured at runtime and explains that metrics can include attributes used to identify dimensions. Derive or record counters for attempts and failures, plus latency distributions, using the same controlled dimensions as the event schema. Do not turn `operation_id` into a metric attribute; it is useful for correlation in logs but creates one value per operation.

Then test boundaries, not pretty output. A useful test matrix covers a normal 2xx completion, a handled 4xx response, an application failure, a response closed before completion, and a double callback. Assert one completion event at most, the final observed status, a nonnegative elapsed duration, and absence of recipient data. Add a contract test that sends a known operation ID through API acceptance, queueing, an attempt, and a terminal update. The join should survive every hop.

Deploy schema changes additively. Readers should tolerate a new field before writers depend on it, and removal should wait until stored queries and cost jobs no longer require it. A log line is an interface once another team bills, alerts, or audits from it. Treat it that way.

## The rejected shortcut still has a valid use case

The rejected option is a single request log that embeds a guessed delivery result and estimated charge. It looks efficient because no join is required. It is wrong for asynchronous notification delivery: the request can finish before the system has a terminal outcome, retries can multiply attempts, and commercial policy can change independently of deployed middleware.

There is a valid smaller case. For a synchronous internal endpoint with no queue, no retry, no delayed callback, and no per-attempt cost, one completion event may contain all relevant evidence. Keep that design while those constraints remain true. The moment delivery crosses an asynchronous boundary, split the events.

The decision is narrow and testable: middleware owns request completion; workers own attempts; outcome consumers own observed delivery state; a downstream attribution job owns monetary calculation. **No component claims evidence it cannot observe.** That rule catches more edge cases than any logger configuration and keeps notification failures explainable under rate limits, delayed outcomes, and retries.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://datatracker.ietf.org/doc/html/rfc5424
