# HLN (Holarchical Logic Network) Technical Specification

## 1. Terminology

### Core Concepts: The Ghost and the Shell
The fundamental unit of HLN is the **Holon**. Every Holon follows the **Ghost-in-the-Shell** pattern:
*   **The Ghost (Internal Logic):** The "mind" of the Holon. It contains the decision logic, state, or the actual task to be performed.
*   **The Shell (Interface):** The "body" of the Holon. It provides a standardized external interface that translates internal results into the RXON protocol. The Shell allows any Ghost to connect to the Grid.

### Holarchy & Transport
*   **Holon:** An autonomous entity (Atomic or Compound) that is both a "whole" (to itself) and a "part" (to the higher-order system).
*   **Shell-Stacking:** The principle of encasing a Compound Ghost (Orchestrator) within a Shell to allow it to be treated as a simple worker by a higher-order system.
*   **RXON (Reverse Axon):** The inter-node communication protocol. A biological metaphor for the "Reverse Nerve" connection initiated by a subordinate Holon (the Shell) towards a superior Holon (the Ghost/Orchestrator).
*   **The Apex (Root):** An Orchestrator that is not encased in a Shell. It serves as the entry point for the User and dispatches tasks to the first layer of Holons.
*   **Blueprint:** A declarative description of a process (a state machine) that an Orchestrator (a Ghost) executes. It defines the sequence of states, the transitions between them, and the contracts (input/output/event schemas) for the tasks dispatched to Holons.

---

## 2. Communication Protocol (RXON)

### 2.1. Connection Model
RXON mandates a **Worker-Initiated** connection model. The Orchestrator **never** connects to the Worker. The Worker "pulls" tasks and "pushes" results/events.

### 2.2. Core Principles
*   **Metadata Everywhere**: Every primary message MUST include an optional `metadata` field for extensibility.
*   **Zero Trust Identity**: Every message SHOULD include a `security` object for end-to-end verification (HMAC SHA256 and Identity Chain).
*   **Message Freshness**: Every protocol message MUST include a `timestamp` (Integer Unix Epoch, in seconds).
*   **Stable Serialization**: Messages MUST be serialized using the **JSON Round-trip** mechanism (canonical key order, standardized number formatting) to ensure signature consistency.

### 2.3. Smart Resource Matching (Smart Matching)
The Orchestrator MUST use the following standardized logic for matching Tasks to Holon Resources:

1.  **Skill Matching**: Matching is performed **strictly** using the `name` field of a Skill.
2.  **Numeric Property Matching (GE Logic)**: Any numeric property (integer or float) found in `properties` dictionaries is matched using **Greater or Equal (>=)** logic.
3.  **Non-numeric Property Matching**:
    - **Strings**: Exact case-sensitive equality.
    - **Lists**: The Holon's value list must **intersect** with the task's required list.
4.  **Hardware Devices**: Matched by `type`. Partial case-insensitive match for `model`.

---

## 3. Data Models (RXON Objects)

### 3.1. Resources
Represents the static capacity and assets of a Holon Shell.
*   **max_concurrent_tasks**: (int, optional) Capacity limit.
*   **devices**: (list of `HardwareDevice`, optional) Dedicated hardware (GPUs, NPUs).
*   **properties**: (dict, optional) Universal dictionary for resource matching.
    *   *Standard Numeric Keys*: `cpu_cores`, `ram_gb`.
    *   *Standard String Keys*: `os`, `arch`, `location`.

### 3.2. ResourcesUsage
Real-time load reporting (used in Heartbeats).
*   **cpu_load_percent**: (float) Current CPU usage (0.0 - 100.0).
*   **ram_used_gb**: (float) Current RAM consumption in Gigabytes.
*   **devices_usage**: (list of `DeviceUsage`, optional) Load and metrics per device.

### 3.3. HardwareDevice
*   **type**: (str) e.g., "gpu", "sensor".
*   **model**: (str, optional) e.g., "NVIDIA A100".
*   **id**: (str, optional) Unique hardware serial/UUID.
*   **properties**: (dict, optional) Device capabilities.
    *   *Standard Numeric Keys*: `vram_gb`, `compute_capability`.

### 3.4. SkillInfo
The contract for a specific ability.
*   **name**: (str) Unique identifier (e.g., "transcribe").
*   **type**: (str, optional) Skill category.
*   **version**: (str, optional) Skill version.
*   **input_schema**: (dict, optional) JSON Schema for task parameters.
*   **output_schema**: (dict, optional) JSON Schema for task results.
*   **events_schema**: (dict, optional) JSON Schema for custom events.
*   **output_statuses**: (list of str, optional) List of custom terminal statuses.
*   **properties**: (dict, optional) Skill metadata (e.g. `{"price": 0.01}`).

### 3.5. SecurityContext
*   **signature**: (str, optional) HMAC-SHA256 hash of the payload.
*   **signer_id**: (str, optional) The `worker_id` of the node that generated the signature.
*   **identity_chain**: (list of str, optional) The path of IDs through which the message bubbled (Ordered: [Origin ID, ..., Previous Hop ID]).
*   **orchestrator_signature**: (str, optional) HMAC-SHA256 signature generated by the Orchestrator using the worker's token, to prevent task injection.

### 3.6. Policy
Execution policy attached to a Job or Task, enforcing autonomy limits.
*   **allowed_skills**: (list of str, optional) Whitelist of skill names the Holon is permitted to execute. If empty or missing, all skills are allowed.
*   **budget_cap**: (float, optional) Maximum cost budget for the job.
*   **Custom fields**: Any additional key-value pairs are passed transitively to sub-jobs and workers.

### 3.7. TaskPayload
The message that carries a unit of work from an Orchestrator to a Holon. It is the envelope a Worker pulls from the queue.
*   **job_id**: (str) The job this task belongs to.
*   **task_id**: (str) Unique identifier of this task (idempotency key for results).
*   **type**: (str) The skill name the Worker must execute.
*   **params**: (dict, optional) Task parameters, validated against the skill's `input_schema`.
*   **policy**: (`Policy`, optional) Execution policy inherited from the job. See §4.3.
*   **holarchy_depth**: (int, optional) Depth of the originating job in the Holarchy. See §4.4.
*   **transition_count**: (int, optional) State-machine transition counter of the originating job. Incremented by the Orchestrator on every local state transition to protect against infinite local loops. Propagated to Workers for tracing and loop detection.
*   **tracing_context**: (dict, optional) W3C trace context passed from Orchestrator to Worker. See §5.1.
*   **priority**: (float, optional) Dispatch priority hint (higher = more urgent).
*   **deadline**: (int, optional) Execution deadline (Unix epoch). Late results MUST be ignored.
*   **security**: (`SecurityContext`, optional) Carries `orchestrator_signature` (see §4.6) and the identity chain.
*   **metadata**: (dict, optional) Free-form extensibility.
*   **timestamp**: (int, optional) Message creation time (Unix epoch, in seconds).

### 3.8. TaskResult
The message a Holon returns to the Orchestrator upon completing (or failing) a task.
*   **job_id**: (str) The job this result belongs to.
*   **task_id**: (str) The task being answered (matches `TaskPayload.task_id`).
*   **worker_id**: (str, optional) The Holon submitting the result.
*   **origin_worker_id**: (str, optional) The original source when the result bubbled through Compound Holons.
*   **status**: (str) Outcome of the task (e.g., `success`, `failure`, `cancelled`, or a skill-defined custom status).
*   **data**: (dict, optional) Result data, validated against the skill's `output_schema`.
*   **error**: (`TaskError`, optional) Structured error when `status` is not `success`.
*   **policy**: (`Policy`, optional) Execution policy returned with the result.
*   **holarchy_depth**: (int, optional) Depth of the originating job in the Holarchy.
*   **transition_count**: (int, optional) State-machine transition counter of the originating job.
*   **costs**: (dict, optional) Expense metrics reported by the Worker (e.g., tokens, compute seconds), used for budget tracking across the Holarchy.
*   **trace_context**: (dict, optional) W3C trace context linking the result back to the job trace. See §5.1.
*   **security**: (`SecurityContext`, optional) Signature and identity chain for this result.
*   **metadata**: (dict, optional) Free-form extensibility.
*   **timestamp**: (int, optional) Message creation time (Unix epoch, in seconds).

### 3.9. WorkerEventPayload
The message a Holon emits to initiate a signal (see [Reactive HLN](concepts/REACTIVE_HLN.md)). It is self-describing, idempotent, and traceable, and may bubble up through the Holarchy.
*   **event_id**: (str) Idempotency key — duplicates MUST be discarded.
*   **worker_id**: (str) The Holon currently emitting the signal.
*   **origin_worker_id**: (str) The original source, preserved across hops.
*   **event_type**: (str) Signal type (e.g., `progress`, `alert`, `signal`).
*   **payload**: (dict) The signal data, validated against the owning Blueprint's `events_schema`.
*   **origin_task_id**: (str, optional) The task that gave rise to the event, if any.
*   **bubbling_chain**: (list of str, optional) Ordered path of Holon IDs the signal has passed through (see §5.2).
*   **target_job_id**: (str, optional) A specific job the event addresses.
*   **target_task_id**: (str, optional) A specific task the event addresses.
*   **trace_context**: (dict, optional) W3C trace context for span linkage. See §5.1.
*   **priority**: (float, optional) Urgency hint.
*   **security**: (`SecurityContext`, optional) Signature and identity chain; the chain is re-signed at each hop.
*   **metadata**: (dict, optional) Free-form extensibility.
*   **timestamp**: (int, optional) When the event was produced (Unix epoch, in seconds).

### 3.10. TaskError
Structured error carried by a `TaskResult` when a task does not succeed.
*   **code**: (str) Machine-readable error code (e.g., `transient`, `permanent`, `invalid_input`, `security`). Determines whether the Orchestrator retries, quarantines, or fails the job.
*   **message**: (str) Human-readable description of the failure.
*   **details**: (dict, optional) Additional diagnostic context.

---

## 4. Security & Integrity

### 4.1. Contract Enforcement
Strict "No Trust" policy:
- **Bi-directional Validation**: Orchestrators MUST validate results against `output_schema`. Workers SHOULD validate tasks against `input_schema`.
- **Authority**: Blueprint schemas always take precedence over Worker declarations.
- **Injection Protection**: Schemas MUST default to `additionalProperties: false`.

### 4.2. Replay Protection
Orchestrators MUST track processed `event_id` and `task_id` within a **60-second** window. Any message with a `timestamp` older than 60s or from the future MUST be rejected.

### 4.3. Policy Enforcement
The Orchestrator MUST enforce execution policies at every dispatch boundary:
*   **Skill Whitelist**: If `policy.allowed_skills` is defined, the Orchestrator MUST reject any task dispatch where the skill name is not in the whitelist. Violations result in `quarantined` status.
*   **Propagation**: Policies MUST be inherited by sub-jobs spawned by a parent job, unless explicitly overridden.
*   **Worker-Side Enforcement**: Workers SHOULD verify `orchestrator_signature` before executing a task, and SHOULD enforce `policy.allowed_skills` locally.

### 4.4. Holarchy Depth Protection
To prevent infinite recursive orchestration (`A → spawns B → spawns A → ...`):
*   Each job carries a `holarchy_depth: int` field (0 for root jobs).
*   When a job spawns a sub-job: `child.holarchy_depth = parent.holarchy_depth + 1`.
*   If `holarchy_depth` reaches the configured maximum (default: 10), the job MUST be quarantined.
*   This is independent of `transition_count`, which protects against local state-machine loops.

### 4.5. Job State Signing (Anti-Tamper)
When cryptographic audit is enabled, the Orchestrator MUST compute an HMAC-SHA256 signature over critical job state fields (`id`, `current_state`, `transition_count`, `holarchy_depth`, `policy`) using a cluster-wide secret key. The signature is stored alongside the job state and verified on every read from the shared state store. Mismatches result in immediate quarantine.

### 4.6. Outbound Task Signing
Before placing a task in the worker queue, the Orchestrator MUST sign the task fields (`job_id`, `task_id`, `transition_count`, `holarchy_depth`, `policy`) using HMAC-SHA256 with the worker's secret token. The signature is placed in `task["security"]["orchestrator_signature"]`. Workers MUST verify this signature before execution.

---

## 5. Observability

### 5.1. Distributed Tracing
Propagation of W3C Trace Context ensures end-to-end visibility:
- **`tracing_context`**: (dict) In `TaskPayload`, used to pass the trace parent from Orchestrator to Worker.
- **`trace_context`**: (dict) In `WorkerEventPayload` AND `TaskResult`, used to link events and final results back to the job trace.
- **Span Hierarchy**: Worker execution spans MUST be children of the Orchestrator's dispatch span.

### 5.2. Bubble-up Visibility
The **`bubbling_chain`** field in events provides a verifiable history of a signal's journey through the Holarchy. The **Apex Orchestrator** has the ultimate visibility into the entire tree of execution.

---

## 6. Middleware Architecture

The Orchestrator supports a pluggable middleware chain that intercepts messages at trust boundaries:

| Hook | Trigger | Purpose |
|---|---|---|
| On handler request | Before local handler execution | Input sanitization, PII masking |
| On handler response | After local handler execution | Output validation, logging |
| On outbound task | Before sending task to worker | Prompt injection detection, PII masking |
| On inbound result | On receiving worker result | Toxicity validation, data demasking |
| On inbound event | On receiving worker event | Event filtering, rate limiting |

*   **Error Policy**: Unhandled middleware exceptions quarantine the job (not `failed`).
*   **Timeout**: Each middleware call is bounded by a timeout (default: 5s).
*   **Order**: Middleware execute in registration order (FIFO).

---

## 7. Observability: D-DNA (Temporal Evidence Chaining)

To protect execution logs from retroactive tampering, each history record contains:
*   **previous_hash**: (str) Hash of the preceding record.
*   **current_hash**: (str) `SHA-256(previous_hash + payload + timestamp)`.
*   The first record's hash is `SHA-256(job_id + secret_salt)` (genesis block).
*   An integrity-verification procedure recalculates the chain over a job's history and reports any breaks.

---
*This specification is maintained by the authors of the HLN Protocol.*
