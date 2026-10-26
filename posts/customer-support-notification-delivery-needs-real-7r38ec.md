# Customer Support Notification Delivery Needs Realtime or Thirty-Second Polling

**Short answer:** use polling every 30 seconds for durable, low-urgency notification counts; use scoped realtime delivery for typing indicators and read-state changes that lose meaning while waiting.

The constraint that decides this choice is not latency alone. It is whether a support agent needs to show a customer an ephemeral state, and whether the browser is allowed to author that state. A 30-second poll is usually fine for durable, low-urgency notification counts. It is a poor fit for a typing indicator, and it does not make a read receipt trustworthy.

For a support inbox, I would split the design: persist notifications and receipts in the database, then deliver state changes over a scoped realtime channel when the interface needs them promptly. The server remains the authority. The client may display a state, but it must not be able to manufacture a receipt for another agent or customer.

## Do support notifications need realtime, or is polling every 30 seconds fine?

A notification such as "three tickets changed" can tolerate a bounded delay if the product makes that delay clear. A typing indicator has a short useful life. By the time a 30-second poll observes it, the person may have sent the message, abandoned the draft, or lost their connection. Read receipts sit between those cases: the visual update is time-sensitive, but the fact that a message was read must survive a reload.

That distinction gives us two clocks. The event clock carries transient presence-like signals, with expiry and reconnect behavior. The storage clock records durable facts such as `read_at`, a conversation identifier, and the actor whose read action was accepted. Mixing the clocks creates familiar bugs: a stale "typing" label that survives a browser sleep, or a green checkmark that appears because an untrusted client sent an arbitrary message ID. I initially treated both as notification delivery, but the recovery rules are different enough that this split is the useful design boundary.

That is the trust boundary.

The relevant question is not "Can this app use realtime?" It is: "Which visible states lose meaning during a 30-second wait?"

## Why token scope matters more than transport

A websocket, server-sent event stream, or long poll can carry bytes. None of them decides who may see those bytes. Authorization has to be evaluated before subscription and again when a durable command changes data.

For a support workspace, I use a token whose claims identify a tenant, a conversation scope, and an actor role. A connection should subscribe to a narrow channel such as a single conversation or an agent's assigned queue, rather than receiving a tenant-wide stream and filtering it in JavaScript. Narrow scope reduces accidental disclosure and makes an audit record understandable.

The server should also reject a receipt whose actor, conversation, or message does not match the authenticated context. The client sends an intent; the server derives the timestamp and records the accepted actor. Client-provided `read_at` values are unnecessary authority.

Here is the shape of that boundary in Python. The transport is deliberately generic; the important part is the authorization decision before the write.

```python
from dataclasses import dataclass
from datetime import datetime, timezone

@dataclass(frozen=True)
class Session:
    tenant_id: str
    actor_id: str
    conversation_id: str

def accept_read(session: Session, message: dict, requested_message_id: str) -> dict:
    if message["id"] != requested_message_id:
        raise PermissionError("message is outside the authorized scope")
    if message["tenant_id"] != session.tenant_id:
        raise PermissionError("tenant mismatch")
    if message["conversation_id"] != session.conversation_id:
        raise PermissionError("conversation mismatch")

    return {
        "message_id": message["id"],
        "actor_id": session.actor_id,
        "read_at": datetime.now(timezone.utc).isoformat(),
    }
```

The production version still needs a database transaction, idempotency, and an authorization lookup that is not represented by this small example. The example is useful because it keeps the trust boundary visible.

## How should reconnects and stale state behave?

Realtime is an optimization for delivery, not the source of truth. On reconnect, the client should authenticate again, resubscribe within its current scope, and fetch a durable cursor or current conversation state. It should not assume that an event missed during a 4-second network gap can be reconstructed from the browser's memory.

Typing signals deserve stricter expiry. Send a start event only while the user is actively composing, refresh it periodically if the product needs that precision, and let the server or client clear it after a short timeout. A stop event is helpful, but it cannot be the only cleanup path: mobile suspension and abrupt tab closure are normal.

Read receipts need idempotent writes. Repeating the same read command after reconnect must produce the same durable outcome, not a second audit fact with a later invented time. If the UI optimistically shows a receipt, it should reconcile with the server response and remove the state when authorization fails. That is where client trust shows up in practice.

## What does the comparison look like in production?

Polling has an honest advantage: it is easy to reason about. One request reads a server snapshot, and a failed request does not leave a hidden subscription behind. It also creates a predictable request cadence, which can be acceptable for a notification badge, administrative queue, or low-urgency digest. Thirty seconds is a product decision, though, not a delivery guarantee; the worst-case wait is near the interval, before request time and scheduling delay are counted.

Realtime has a different failure surface. Connections expire, permissions change while a tab is open, proxies interrupt idle streams, and a reconnect can duplicate events. The remedy is not to promise perfect delivery. Use sequence numbers or a durable cursor for replayable events, make handlers idempotent, and expose metrics for authorization rejects, reconnects, event age, and receipt-write latency. A dashboard that reports only connected clients hides the failures users actually feel. The limitation is operational: realtime is the wrong fit for a team that cannot own reconnect, replay, authorization changes, and stale-state cleanup. In that case, a slower poll with an explicit freshness label is the more honest choice.

A hybrid is often the cleanest decision for customer support. Poll durable counts every 30 seconds if that meets the service objective; use a scoped stream for typing and read-state changes; reload authoritative conversation state after reconnect. This keeps the high-frequency path small while preserving a boring recovery path. Boring is good here.

I would test the decision with a small matrix: a backgrounded mobile tab, a token expiring during composition, a reassigned conversation, a duplicated read command, and a reconnect after an event was emitted. The acceptance condition is not merely that an event arrives. It is that an unauthorized client cannot observe it, a missed event cannot erase durable truth, and a stale transient state eventually disappears.

## A compact rollout rule

Start by labeling every notification as durable or ephemeral, then assign each label a maximum acceptable staleness. If the interval is comfortably below that limit, polling is a valid choice. If the interval makes a typing indicator meaningless, or makes an agent act on stale customer state, use realtime delivery for that state.

Keep the authorization model independent from the transport so the decision can change later. Ship the durable write and reconnect read first, add event delivery second, and instrument both paths before increasing fan-out. The safest default for this support workflow is a hybrid: periodic checks for durable, low-urgency notifications; scoped realtime events for short-lived interaction state; server-owned receipts for anything that becomes part of the record.

## Sources

References:

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- MDN, WebSocket API: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
- MDN, Server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- RFC 6455, The WebSocket Protocol: https://www.rfc-editor.org/rfc/rfc6455
