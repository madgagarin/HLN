# Reactive HLN: Worker-Initiated Events

> **"The muscle speaks. The brain listens. The signal climbs."**

In its baseline form, HLN is *command-driven*: the Orchestrator pushes intent downward, and Holons report results upward. **Reactive HLN** completes the model by letting any Holon — most often an Atomic one close to the physical world — *initiate* a signal. The Holarchy becomes bidirectional: commands flow down, and meaningful signals bubble up until they reach a layer intelligent enough to act on them.

---

## 1. Why Reactive?

A purely command-driven network assumes the Orchestrator always knows when to ask. That breaks down for:

*   **Physical Holons**: a sensor should not wait to be polled to report that a motor is overheating.
*   **Long tasks**: a worker should be able to emit progress, partial findings, or alerts without a pending question.
*   **Emergent intent**: an Atomic Holon may discover something the Orchestrator never planned for.

Reactive HLN turns the Holon from a passive executor into an **active participant**, while keeping the Orchestrator as the single source of truth for *what to do next*.

---

## 2. The Event Object

Every signal is carried by a `WorkerEventPayload`. It is a self-describing, idempotent, traceable message.

```python
class WorkerEventPayload(NamedTuple):
    event_id: str            # Idempotency key — duplicates are discarded
    worker_id: str           # The Holon currently emitting the signal
    origin_worker_id: str    # The original source (preserved across hops)
    event_type: str          # "alert", "signal", "progress", "log", ...
    payload: dict            # The signal data
    origin_task_id: str      # The task that gave rise to the event (if any)
    bubbling_chain: list[str]# The verifiable path the signal has travelled
    target_job_id: str       # Optional: a specific job the event addresses
    target_task_id: str      # Optional: a specific task the event addresses
    trace_context: dict      # W3C trace context for span linkage
    priority: float          # Urgency hint for dispatch
    security: dict           # Signature and identity chain
    metadata: dict           # Free-form extensibility
    timestamp: int           # When the event was produced (Unix epoch)
```

Two fields make the event trustworthy across an unbounded Holarchy:

*   **`origin_worker_id`** never changes, so the *true* source is always identifiable even after the signal has bounced through several Compound Holons.
*   **`bubbling_chain`** is the ordered list of Holon IDs the signal has passed through, and is cryptographically covered by the `SecurityContext`. It cannot be forged.

---

## 3. Signaling and Bubbling

### Signaling
When an event arrives at an Orchestrator, it is matched against the **active contexts** of the jobs that currently run there. If a running job is the intended target (or is logically responsible for the emitting worker), the event is *delivered* into that job's context — where a Blueprint handler can react to it, the same way it reacts to a task result.

### Bubbling
If the current Orchestrator has **no** active context that can handle the event, the event is not dropped — it *bubbles up* to the parent Orchestrator (the Compound Holon one level higher). The bubbling repeats recursively:

```
Atomic Holon → Compound Holon (Layer N) → ... → Apex
```

At each hop:

1. The forwarding Holon appends its own ID to `bubbling_chain`.
2. The `SecurityContext` is re-signed so the extended chain remains verifiable.
3. `trace_context` is preserved so the whole journey forms one trace.

Bubbling stops when an Orchestrator **finds a handler** for the event, or when the **Apex** is reached. The Apex has ultimate visibility into the entire tree of execution, so even an unhandled event is observable at the top.

---

## 4. Backpressure and the Dead Letter Queue

A reactive network must protect itself from a flood of signals.

*   **Backpressure**: the Orchestrator may answer an event with an `EventResponse` carrying directives — for example, telling a worker to throttle a given `event_type`, to silence it for a cooldown, or to keep sending. The worker is expected to honor these directives locally.
*   **Dead Letter Queue (DLQ)**: events that cannot be delivered, cannot be matched, or violate the schema are not silently lost — they are moved to a DLQ for later inspection. This makes signal loss an *observable, recoverable* condition rather than a silent bug.

---

## 5. Auditability

Reactive activity is part of the execution record. The history layer records the lifecycle of every event:

*   **Arrival**: the fact that an event reached an Orchestrator, with its full metadata.
*   **Trigger**: when an event causes a new job to start, the link between the `event_id` and the new `job_id` is preserved.
*   **Bubbling**: each hop (`from_orchestrator → to_orchestrator`) is recorded, so the path of a signal through the Holarchy can be reconstructed after the fact.

Combined with the **D-DNA** temporal chain (see [Security Extensions](SECURITY_EXTENSIONS.md)), the reactive record is tamper-evident: a signal's journey, once written, cannot be rewritten without breaking the hash chain.

---

## 6. Observability

Reactive HLN is a first-class citizen of the observability model:

*   **Metrics**: the flow of events (received, rejected, bubbled), bubbling-depth histograms, and end-to-end event-processing latency are all exposed alongside the command-side metrics.
*   **Tracing**: an event that triggers a new job is linked to that job via a **Span Link** (the event and the job may have different lifecycles, so a strict parent-child span would be misleading). The "ascent" of a signal through the layers is rendered as a sequence of nested spans, making the bubble path visible in any W3C-aware backend.

---

## 7. Edge Intelligence

To keep the network clean, intelligence is pushed to the edge:

*   **Edge Filtering**: a Holon filters its own noise according to rules provided by the Orchestrator, so only meaningful signals ever travel up.
*   **Local Buffer**: if the connection to the Orchestrator is temporarily lost, the Holon buffers events locally and delivers them on reconnect, preserving order and idempotency.
*   **Webhook Gateway**: a Holon can act as an ingress point for external HTTP signals, encapsulating them as `WorkerEventPayload` objects so the rest of the Holarchy sees only native RXON events.

---

## 8. Security Guarantees

Reactive does not mean uncontrolled. The same Zero Trust rules that govern commands govern signals:

*   **Idempotency**: `event_id` is checked for duplicates, so a replayed or re-delivered event is processed exactly once.
*   **Quotas**: every worker is subject to per-type event-rate limits; a noisy or compromised Holon cannot drown the Orchestrator.
*   **Schema Validation**: the `payload` of an event is validated against the schema declared by the Blueprint that owns the `event_type`. Malformed events are rejected and routed to the DLQ.
*   **Identity Chain**: the `bubbling_chain` is signed at every hop, so the reported path of a signal is verifiable end to end.

---

## 9. Related Documents

- **HLN Specification**: `SPECIFICATION.md` (Section 5 — Observability, and Section 4 — Security & Integrity).
- **Distributed Tracing**: `concepts/DISTRIBUTED_TRACING.md`.
- **Security Extensions**: `concepts/SECURITY_EXTENSIONS.md` (D-DNA and Identity Chain).

---

*This document is part of the HLN architectural specification.*
