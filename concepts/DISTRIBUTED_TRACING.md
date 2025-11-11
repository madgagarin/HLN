# Distributed Tracing in HLN Holarchy

To maintain visibility across layers of the HLN, distributed tracing is essential. It allows a user to track a request from the initial "Apex" Orchestrator down to the deepest "Atomic Holon."

## Trace Context Propagation

HLN follows the **W3C Trace Context** standard. Every message in the RXON protocol that initiates a process or task MUST include a `tracing_context` field.

### 1. The Context Object
The `tracing_context` is a dictionary containing standard headers, primarily `traceparent` (and optionally `tracestate`).

```json
"tracing_context": {
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

## 🪜 Propagation Layers

### Layer 1: Apex to Compound Holon
1.  The **Apex Orchestrator** creates a root span for a new Job.
2.  When it dispatches a task to a **Compound Holon**, it injects the current trace context into the `TaskPayload`.
3.  The **Holon Shell** receives the task. It extracts the context and uses it as the parent for the internal Job it is about to start.

### Layer 2: Inside the Matryoshka
1.  The **Internal Ghost** (Orchestrator) starts its own spans, linked to the parent context from the Shell.
2.  If this Ghost dispatches tasks to its own sub-workers, the process repeats.

### Layer 3: Atomic Holon (The Worker)
1.  A **Worker** receives a `TaskPayload`.
2.  It automatically sets up a local span for the task execution using the provided `tracing_context`.
3.  Any logs or sub-calls (e.g., to an LLM API) made within the task are tagged with this trace ID.

## Span Links and Parent-Child Relationships

*   **Primary Relationship:** Each layer uses the incoming context as a **Parent Span**. This creates a single, continuous trace tree.
*   **Recursive Signaling:** In cases where a worker initiates an event (Reactive HLN), a **Span Link** SHOULD be used to connect the event to the job it eventually triggers, as the event and the job might have different lifecycles.

## Reference Implementation Notes

A conforming implementation propagates the trace context as follows:
*   **Orchestrator**: instruments its listener and internal job execution with OpenTelemetry.
*   **Compound Holon (Shell-Stacking bridge)**: extracts the context from an incoming task and passes it to the internal job it starts.
*   **Worker**: injects the context into the handler so the developer can propagate it further into downstream calls.

## 👁 Observability Benefits

By strictly propagating `tracing_context`, HLN enables:
1.  **Latency Analysis**: Pinpointing which layer of the holarchy is causing delays.
2.  **Error Root-Cause**: Seeing the exact worker error that caused a cascade of failures up to the Apex.
3.  **Cost Attribution**: Linking compute resources consumed at the bottom to the specific user request at the top.
