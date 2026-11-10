# HLN (Holarchical Logic Network) Technical Specification

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Conformance:** RFC 2119 / RFC 8174  

---

## 1. Terminology & Core Architecture

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here.

### 1.1. The Ghost and the Shell
The fundamental building block of an HLN network is the **Holon**. Every Holon decouples internal execution logic from the external network contract:
*   **The Ghost (Execution Logic):** The internal logic of the Holon. It contains domain business logic, state machines, machine learning models, scripts, physical hardware controllers, or human operators.
*   **The Shell (Network Adapter):** The standardized external interface that bridges the Ghost into the HLN Grid using the **RXON** protocol. The Shell enables any Ghost to participate in the network through an identical contract.

### 1.2. Holarchy & Fractal Composition
*   **Holon:** An autonomous entity that is simultaneously a self-contained whole to its sub-components and a dependent part of a higher-order system (Arthur Koestler, 1967).
*   **Atomic Holon:** A leaf worker executing a single function, skill, or hardware interface without subordinates.
*   **Compound Holon (Shell-Stacking):** An entire Orchestrator managing its own internal holarchy, encased inside a Shell. To a higher-order Orchestrator, this Compound Holon appears as a standard single worker.
*   **The Apex (Root):** An Orchestrator that is not encased in a Shell. It serves as the primary ingress point for external clients and dispatches tasks to Layer 1 Holons.
*   **Blueprint:** A declarative Finite State Machine (FSM) defining states, transitions, contracts, deciders, and schemas executed by an Orchestrator. See [Blueprint Specification](blueprint.md).
*   **RXON (Reverse Axon):** The inter-node communication protocol enforcing a strictly worker-initiated (pull) network topology.

---

## 2. Communication Protocol (RXON)

### 2.1. Connection Topology: The Pull Model
RXON mandates a **Worker-Initiated (Pull)** connection model:
1. The Orchestrator **MUST NOT** initiate inbound network connections to Workers.
2. The Worker **MUST** initiate all connections upstream to claim tasks and report results/events.
3. Workers operate behind NATs, corporate firewalls, and on edge IoT hardware without open listening ports.

### 2.2. HTTP Transport Binding
In the standard HTTP/1.1 and HTTP/2 transport mapping, communication between Workers and Orchestrators adheres to the following endpoints (prefixed by the worker API root, default `/_worker`):

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/_worker/workers/register` | Worker registration announcing skills, resources, and credentials. |
| `PATCH` | `/_worker/workers/{worker_id}` | Periodic heartbeat reporting load, active tasks, and receiving control directives. |
| `POST` | `/_worker/workers/{worker_id}/tasks/claim` | Atomic task claiming under lease (Long-Polling). |
| `GET` | `/_worker/workers/{worker_id}/tasks/next` | *(Deprecated)* Read-only polling alias maintained for backwards compatibility. |
| `POST` | `/_worker/tasks/result` | Submission of completed task result or error payload. |
| `POST` | `/_worker/events` | Worker-initiated event signaling (progress, alert, algedonic signals). |
| `POST` | `/_worker/auth/token` | STS token exchange endpoint (mTLS client certificate to short-lived token). |

#### 2.2.1. Task Claim Semantics & Long-Polling
- The task acquisition endpoint is **`POST /_worker/workers/{worker_id}/tasks/claim`** because claiming a task mutates queue state and assigns an exclusive lease.
- When no tasks are available in the queue matching the worker's capabilities, the Orchestrator **MUST** hold the connection open (long-poll) up to `timeout` seconds (RECOMMENDED: 30s).
- If the long-poll timeout expires without a task becoming available, the Orchestrator **MUST** return **`204 No Content`** with an empty response body.

#### 2.2.2. HTTP Status Codes & Error Handling
Orchestrators **MUST** return HTTP status codes with semantic fidelity:

| Status Code | Meaning & Usage |
|---|---|
| `200 OK` | Task claimed successfully, or result/event accepted. |
| `204 No Content` | Long-poll expired with empty queue (no tasks available). |
| `400 Bad Request` | Malformed JSON or JSON Schema contract validation failure. |
| `401 Unauthorized` | Missing or invalid authentication token / expired STS credential. |
| `403 Forbidden` | Policy violation, audience mismatch, or worker ID spoofing. |
| `409 Conflict` | Fencing token conflict (`attempt` or `job_epoch` superseded), or duplicate non-idempotent event. |
| `410 Gone` | Task deadline expired, lease revoked, or job cancelled; worker **MUST NOT** retry result submission. |
| `429 Too Many Requests` | Rate limit or quota exceeded; **MUST** include `Retry-After` header. |
| `503 Service Unavailable` | Orchestrator draining, shutting down, or backend unavailable; **MUST** include `Retry-After` header. |

When returning `4xx` or `5xx` errors, the response body **MUST** conform to the structured `ErrorResponse` schema:
```json
{
  "error": "Brief machine-readable error string",
  "code": "ERROR_CODE",
  "details": {},
  "retry_after_s": 30
}
```

#### 2.2.3. Backoff, Jitter, and Rate-Limiting Dynamics
Workers **MUST** implement robust backoff dynamics to eliminate thundering herd storms during orchestrator restarts:
1. **Exponential Backoff with Full Jitter:** When network errors or `503 Service Unavailable` occur, backoff delay $t$ **MUST** be calculated as:
   $$t = \text{random}(0,\ \min(t_{\text{max}},\ t_{\text{initial}} \cdot 2^{\text{consecutive\_failures}}))$$
   where $t_{\text{initial}} = 1.0\text{s}$ and $t_{\text{max}} = 60.0\text{s}$.
2. **`Retry-After` Parsing:** If a response contains a `Retry-After` header (represented either as an integer number of seconds or an HTTP-date per RFC 9110), the worker **MUST** pause polling for that endpoint for at least the specified duration before attempting another request.
3. **Heartbeat Debounce & Cooldown:** Heartbeats **MUST** enforce a minimum cooldown interval between consecutive dispatches (RECOMMENDED: 1.0s) to prevent flood during high-frequency local events.

---

### 2.3. Directional Resource & Skill Matching

Matching tasks to candidate Holons **MUST NOT** apply universal greater-or-equal (GE) logic to all numeric fields, which causes inverted matching on cost and latency.

The Orchestrator **MUST** apply **Directional Semantics**:

1. **Skill Matching:**
   - Evaluated strictly using the `name` field of `SkillInfo`.
   - If `version` is specified, it **MUST** match exactly.

2. **Directional Numeric Matching:**
   Numeric properties in requirements are evaluated according to their semantic direction:
   - **Floors (Capacity Minimums):** Properties representing available capacity (`cpu_cores`, `ram_gb`, `vram_gb`, `disk_gb`, `compute_capability`) require that the worker's reported value is **greater than or equal to** the requirement ($\text{worker\_val} \ge \text{req\_val}$).
   - **Ceilings (Cost and Latency Maximums):** Properties representing costs or latencies (`price`, `cost_per_token`, `cost_per_task`, `latency_ms`, `ping_ms`) require that the worker's reported value is **less than or equal to** the requirement ($\text{worker\_val} \le \text{req\_val}$).
   - **Explicit Operators:** Tasks **MAY** specify explicit operator dictionaries:
     ```json
     {"vram_gb": {"$gte": 24}, "price": {"$lte": 0.05}, "region": {"$in": ["us-east-1", "eu-central-1"]}}
     ```

3. **List & Capability Matching (Superset Requirement):**
   When a task requires a list of capabilities or features, the worker's capability list **MUST be a superset** of the required list ($\text{req\_list} \subseteq \text{worker\_list}$). Mere non-empty intersection is non-compliant and **PROHIBITED**.

4. **Hardware Devices:**
   Matched by `type` (exact match) and `model` (case-insensitive substring match).

---

## 3. Data Models (RXON Protocol Objects)

All data structures in this section **MUST** use RFC 8785 (JCS) canonical serialization when hashed or signed.

### 3.1. Resources
Represents the static capacity of a Holon:
*   `cpu_cores`: (int, optional) Number of logical CPU cores.
*   `ram_gb`: (float, optional) System RAM capacity in Gigabytes.
*   `devices`: (list of `HardwareDevice`, optional) Dedicated hardware accelerators.
*   `properties`: (dict, optional) Universal dictionary for resource matching (`region`, `location`, `os`, `arch`).

### 3.2. ResourcesUsage
Real-time load telemetry reported in heartbeats:
*   `cpu_load_percent`: (float) Current CPU utilization (0.0 – 100.0).
*   `ram_used_gb`: (float) Current RAM consumption in Gigabytes.
*   `devices_usage`: (list of `DeviceUsage`, optional) Per-device load metrics.

### 3.3. DeviceUsage
*   `unit_id`: (str) Hardware unit identifier (e.g., "gpu:0").
*   `load_percent`: (float) Device compute load percentage.
*   `vram_used_gb`: (float, optional) VRAM currently consumed.
*   `metrics`: (dict, optional) Diagnostic telemetry (temperature, power watts).

### 3.4. HardwareDevice
*   `type`: (str) Device category ("gpu", "npu", "fpga", "sensor", "actuator").
*   `model`: (str, optional) Hardware model designation (e.g., "NVIDIA RTX 4090").
*   `id`: (str, optional) Hardware serial number or UUID.
*   `properties`: (dict, optional) Device capabilities:
    *   *Standard Keys*: `vram_gb` (float), `compute_capability` (str).

### 3.5. SkillPricing
*   `cost_per_token`: (float, optional) Token-based pricing rate.
*   `cost_per_task`: (float, optional) Flat invocation charge.
*   `currency`: (str, default: "USD") ISO 4217 currency code or internal credit unit.

### 3.6. SkillInfo
The contract for a registered capability:
*   `name`: (str) Unique skill identifier (e.g., "transcribe").
*   `type`: (str, optional) Category grouping.
*   `version`: (str, optional) Semantic version.
*   `input_schema`: (dict, optional) JSON Schema Draft 2020-12 for input parameters.
*   `output_schema`: (dict, optional) JSON Schema Draft 2020-12 for result data.
*   `events_schema`: (dict, optional) JSON Schema for custom events emitted by this skill.
*   `output_statuses`: (list of str, optional) Supported custom terminal statuses.
*   `pricing`: (`SkillPricing`, optional) Standardized cost structure.
*   `properties`: (dict, optional) Metadata.

### 3.7. SecurityContext
Carries cryptographic authentication and provenance:
*   `signature`: (str, optional) Signature of the message payload.
*   `signer_id`: (str, optional) Authenticated ID of the signing node.
*   `orchestrator_signature`: (str, optional) Signature over the entire task envelope.
*   `provenance_chain`: (list of dict, optional) Append-only list of hop signatures.
*   `kid`: (str, optional) Key identifier for STS token or public key lookup.

### 3.8. Policy
Constrains the execution perimeter of a Job or Task:
*   `allowed_skills`: (list of str, optional) Whitelist of permitted skill names. If omitted or empty, all registered skills are allowed.
*   `budget_cap`: (float, optional) Maximum aggregate expenditure allowed for the job.
*   `budget_allocated`: (float, optional) Specific budget slice allocated to this task/child.
*   `budget_currency`: (str, default: "USD") Currency unit for budget enforcement.
*   `tenant_id`: (str, optional) Multi-tenant tenant boundary identifier.
*   `data_classification`: (str, optional) Security classification ("public", "internal", "confidential", "restricted").

### 3.9. TaskPayload
The complete unit of work claimed by a Worker:
*   `job_id`: (str) Unique identifier of the owning job.
*   `task_id`: (str) Unique identifier of this specific task execution.
*   `attempt`: (int, default: 1) Monotonically increasing attempt counter for this task.
*   `job_epoch`: (int, default: 1) Fencing epoch of the owning job.
*   `type`: (str) Skill name to execute.
*   `params`: (dict, optional) Parameters validated against the skill's `input_schema`.
*   `params_sha256`: (str, optional) Checksum for large payloads referenced via external URI (`blob://`).
*   `policy`: (`Policy`, optional) Attenuated execution policy.
*   `holarchy_depth`: (int, default: 0) Depth in the holarchy hierarchy.
*   `transition_count`: (int, default: 0) Transition counter of the originating job.
*   `trace_context`: (dict, optional) W3C trace context (`traceparent`, `tracestate`).
*   `priority`: (float, default: 0.0) Priority scheduling weight.
*   `created_at`: (int) Source creation timestamp (Unix epoch milliseconds).
*   `envelope_ts`: (int) Network packet dispatch timestamp (Unix epoch milliseconds).
*   `nonce`: (str) Unique cryptographic nonce / UUIDv7 for envelope replay protection.
*   `deadline`: (int) Hard task execution deadline (Unix epoch milliseconds).
*   `schedule_to_start_timeout_s`: (float, optional) Queue residency timeout.
*   `start_to_close_timeout_s`: (float, optional) Maximum execution timeout post-claim.
*   `lease_ttl_s`: (int, default: 60) Duration of task lease granted to claiming worker.
*   `orphan_policy`: (str, default: "abort") Behavior if upstream connection is severed ("abort", "suspend", "continue_bounded").
*   `autonomy_ttl_s`: (int, default: 0) Bounded autonomy duration if `orphan_policy` is "continue_bounded".
*   `retry_budget`: (int, default: 3) Remaining retry budget for this subtree.
*   `target_worker_id`: (str) Audience binding identifier.
*   `security`: (`SecurityContext`, optional) Contains `orchestrator_signature`.
*   `metadata`: (dict, optional) Extensible metadata.

### 3.10. TaskResult
The outcome returned by a Worker upon completing or failing a task:
*   `job_id`: (str) Matches `TaskPayload.job_id`.
*   `task_id`: (str) Matches `TaskPayload.task_id`.
*   `attempt`: (int) Matches the claimed task `attempt`.
*   `job_epoch`: (int) Matches the claimed task `job_epoch`.
*   `worker_id`: (str) ID of the Holon submitting the result.
*   `origin_worker_id`: (str, optional) Root source ID if bubbled through Compound Holons.
*   `status`: (str) Execution outcome (`success`, `failure`, `cancelled`, or skill custom status).
*   `data`: (dict, optional) Output payload validated against skill's `output_schema`.
*   `error`: (`TaskError`, optional) Structured error description if status is not `success`.
*   `costs`: (dict, optional) Expense metrics reported by worker (tokens, compute_seconds).
*   `trace_context`: (dict, optional) OpenTelemetry trace context.
*   `created_at`: (int) Result creation timestamp (Unix epoch milliseconds).
*   `envelope_ts`: (int) Envelope dispatch timestamp (Unix epoch milliseconds).
*   `nonce`: (str) Replay prevention nonce.
*   `security`: (`SecurityContext`, optional) Cryptographic signature of the result.
*   `metadata`: (dict, optional) Extensible metadata.

> [!WARNING]
> `TaskResult` **MUST NOT** include an authoritative `policy` field. Any `policy` returned in a result payload **MUST BE IGNORED** by the Orchestrator to prevent policy elevation attacks.

### 3.11. WorkerRegistration
Announced by a Worker upon registration:
*   `worker_id`: (str) Unique persistent identifier of the worker.
*   `worker_type`: (str, default: "generic") Classification ("gpu-worker", "compound-holon", "human-console").
*   `supported_skills`: (list of `SkillInfo`) Complete contracts for all supported skills.
*   `resources`: (`Resources`) Static hardware and property assets.
*   `max_concurrent_tasks`: (int, default: 1) Capacity limit.
*   `installed_software`: (dict, optional) Software runtime versions (`{"python": "3.12"}`).
*   `capabilities`: (dict, optional) Network hostname, IP, and affinity properties.
*   `public_key_pem`: (str, optional) Ed25519 public key (Profile L2).
*   `protocol_version`: (str, default: "1.2.0") Preferred RXON protocol version.
*   `protocol_min_version`: (str, default: "1.0.0") Minimum acceptable protocol version.
*   `protocol_max_version`: (str, default: "1.2.0") Maximum acceptable protocol version.
*   `created_at`: (int) Registration timestamp.

### 3.12. HeartbeatPayload & HeartbeatResponse
**Request (`HeartbeatPayload`):**
*   `worker_id`: (str) ID of the heartbeating worker.
*   `status`: (str) Operational status ("active", "idle", "busy", "draining").
*   `usage`: (`ResourcesUsage`) Real-time compute load.
*   `current_tasks`: (list of str) IDs of currently executing tasks (extends active leases).
*   `available_skills`: (list of str, optional) Dynamic skill readiness.
*   `hot_skills`: (list of str, optional) In-memory warm models/contexts for affinity routing.
*   `mounted_manifests`: (list of str, optional) Manifest digests currently mounted on the worker.
*   `available_credits`: (int, optional) Backpressure credit reporting.
*   `envelope_ts`: (int) Envelope timestamp.

**Response (`HeartbeatResponse`):**
*   `status`: (str) "ok".
*   `cancel_requested`: (list of str, optional) Array of `task_id` values marked for cooperative cancellation.
*   `mount_digests`: (list of str, optional) Manifest digests the worker must download from CAS and mount.
*   `unmount_digests`: (list of str, optional) Manifest digests the worker must drain and unmount.
*   `require_full_sync`: (bool, default: false) Signals worker to re-register.
*   `drain_requested`: (bool, default: false) Signals worker to gracefully drain.

### 3.13. WorkerEventPayload & EventResponse
**Event (`WorkerEventPayload`):**
*   `event_id`: (str) Unique idempotency identifier (UUIDv7 recommended).
*   `worker_id`: (str) Immediate forwarding worker ID.
*   `origin_worker_id`: (str) Original source worker ID.
*   `event_type`: (str) Signal classification ("progress", "alert", "signal").
*   `payload`: (dict) Business event data validated against Blueprint `events_schema`.
*   `origin_task_id`: (str, optional) Task that gave rise to this event.
*   `bubbling_chain`: (list of str) Chronological path of worker IDs traversed.
*   `provenance_chain`: (list of dict, optional) Append-only cryptographic hop signatures.
*   `bubbling_ttl`: (int, default: 5) Decrementing hop limit to prevent endless routing loops.
*   `target_job_id`: (str, optional) Targeted job ID.
*   `target_task_id`: (str, optional) Targeted task ID.
*   `trace_context`: (dict, optional) W3C trace context.
*   `created_at`: (int) Timestamp of signal generation at origin (ms).
*   `envelope_ts`: (int) Timestamp of this transmission envelope (ms).
*   `nonce`: (str) Replay prevention nonce.
*   `security`: (`SecurityContext`, optional) Cryptographic signature.

**Response (`EventResponse`):**
*   `status`: (str) "accepted" | "routed" | "dropped".
*   `throttle_ms`: (int, default: 0) Backpressure directive requesting worker to delay future events.
*   `drop_types`: (list of str, optional) Signal types currently suppressed by orchestrator.

### 3.14. TaskError
Structured error object:
*   `code`: (str) Error code (`TRANSIENT_ERROR`, `PERMANENT_ERROR`, `INVALID_INPUT_ERROR`, `SECURITY_ERROR`, `TIMEOUT_ERROR`, `RESOURCE_EXHAUSTED_ERROR`, `CONTRACT_VIOLATION_ERROR`).
*   `message`: (str) Human-readable explanation.
*   `details`: (dict, optional) Diagnostic stack traces or field validation errors.

---

## 4. Distributed Execution Reliability & Security

### 4.1. Three-Field Model of Time & Freshness
A single `timestamp` field conflates creation time, queue latency, offline buffering, and network freshness.

HLN mandates three distinct temporal attributes:
1. **`created_at` (Business Time):** Set once at origin creation; remains unchanged in offline buffers.
2. **`envelope_ts` + `nonce` (Transport Freshness):** Set afresh for every network packet transmission. Freshness validation applies **only to `envelope_ts`**:
   $$|\text{now} - \text{envelope\_ts}| \le \text{max\_clock\_skew\_s}\quad (\text{RECOMMENDED: } 300\text{ seconds})$$
3. **`deadline` (Execution Timeout):** Absolute wall-clock expiration timestamp (Unix epoch milliseconds).
   - **Clamping Formula for Compound Holons:** When a Compound Holon dispatches a child task, the child deadline **MUST** be clamped:
     $$\text{child\_deadline} = \min(\text{now} + \text{step\_timeout},\ \text{parent\_deadline} - \text{propagation\_reserve})$$
     where `propagation_reserve` ensures sufficient time for result assembly and bubbling.

### 4.2. Lease Management, Fencing Tokens & Split-Brain Prevention
1. **Task Lease:** Task claim grants a lease of `lease_ttl_s` (default: 60s). The worker **MUST** renew this lease by including the `task_id` in `current_tasks` in each periodic heartbeat.
2. **Fencing Tokens (`job_epoch`, `attempt`):**
   - Every task dispatch carries monotonically increasing integer tokens `(job_epoch, attempt)`.
   - If a lease expires without renewal, the orchestrator reassigns the task with `attempt = attempt + 1`.
   - If an orphaned worker subsequently submits a result with an obsolete `attempt` or `job_epoch`, the orchestrator **MUST** reject or ignore the submission (e.g., HTTP `200 OK` with `{"status": "ignored", "reason": "stale_attempt"}`, or `409 Conflict` / `410 Gone`).
3. **Orphan Policy (`orphan_policy`):**
   - `abort` (Default for compute): If worker loses upstream communication for $> \text{autonomy\_ttl\_s}$, it aborts execution to prevent budget waste.
   - `continue_bounded` (Physical / edge loops): Continues execution for up to `autonomy_ttl_s`, followed by state reconciliation upon reconnection.

### 4.3. Cooperative Cancellation & Credit-Based Backpressure
1. **Cancellation Directives:** Orchestrators include `cancel_requested: [task_id, ...]` in responses to `PATCH .../heartbeat` and `POST .../tasks/claim`.
2. **Worker Interruption:** Upon detecting its active task in `cancel_requested`, the worker aborts execution and submits `TaskResult` with `status = "cancelled"`.
3. **Cascading Teardown:** When a Compound Holon receives cancellation for its task, it propagates cancellation to all active child jobs according to its configured `parent_close_policy` (`TERMINATE`, `REQUEST_CANCEL`, `ABANDON`).
4. **Credit Backpressure:** Compound Holons publish `available_credits` (or dynamically adjusted `max_concurrent_tasks`) calculated as the min-cut capacity of their worker subtree.

### 4.4. Budgets & Monotonic Policy Attenuation
1. **Divisible Budgets (`budget_allocated`):** A parent job's budget is divided into allocated reserves. The sum of child budgets **MUST NOT** exceed the parent's unallocated reserve ($\sum \text{budget}_{\text{child}} \le \text{budget\_remaining}_{\text{parent}}$).
2. **Monotonic Policy Attenuation:** Sub-jobs can **only narrow** permissions:
   $$\text{allowed\_skills}_{\text{child}} \subseteq \text{allowed\_skills}_{\text{parent}}$$
   $$\text{budget}_{\text{child}} \le \text{budget\_remaining}_{\text{parent}}$$

### 4.5. Holarchy Depth Protection
- Each job maintains an integer `holarchy_depth` (0 for root jobs).
- Spawned sub-jobs receive `child.holarchy_depth = parent.holarchy_depth + 1`.
- If `holarchy_depth > max_holarchy_depth` (default: 10), the job **MUST** be quarantined.
- If a reactive event triggers a new Job, it **MUST NOT** reset depth to 0; it inherits `holarchy_depth + 1` from the originating context.

### 4.6. Full Envelope Cryptographic Coverage
Under both Profile L1 and Profile L2, signatures **MUST** cover the canonical RFC 8785 representation of the full envelope:
- **`TaskPayload` Signature:** Covers `job_id`, `task_id`, `attempt`, `job_epoch`, `type`, `params` (or `params_sha256`), `deadline`, `priority`, `policy`, `holarchy_depth`, `transition_count`, `trace_context`, `created_at`, `envelope_ts`, `nonce`, and `target_worker_id`.
- **`JobState` Signature:** Covers `id`, `state_version`, `current_state`, `transition_count`, `holarchy_depth`, `policy`, `budget_spent`, queue hash, and `d_dna_head_hash`. State writes **MUST** use Compare-And-Swap (CAS) on `state_version` to prevent rollback.

### 4.7. Append-Only `provenance_chain`
Destructive re-signing of `bubbling_chain` is **PROHIBITED**. Each intermediate forwarding hop $i$ **MUST** append an entry to `provenance_chain`:
$$\text{sig}_i = \text{Sign}_{K_i}(\text{JCS}(\{\text{prev\_sig}: \text{sig}_{i-1},\ \text{hop\_id}: \text{id}_i,\ \text{ts}: t_i,\ \text{payload\_hash}: H(P)\}))$$

---

## 5. Observability: D-DNA Audit Chaining

To provide tamper-evident execution logs, each state transition records an event verified via keyed HMAC with sequence numbers:
$$\text{record\_mac}_k = \text{HMAC-SHA256}\left(K_{\text{audit}},\ \text{JCS}(\{\text{job\_id},\ \text{seq}: k,\ \text{prev\_mac}: \text{record\_mac}_{k-1},\ \text{ts},\ \text{event\_type},\ \text{payload\_hash}\})\right)$$
where:
- $\text{record\_mac}_0 = \text{HMAC-SHA256}(K_{\text{audit}},\ \text{JCS}(\{\text{job\_id},\ \text{seq}: 0,\ \text{prev\_mac}: \text{""},\ \text{salt}\}))$.
- `seq` is a strictly monotonic integer sequence number ($0, 1, 2, \dots$).
- Canonical JSON is formatted strictly per RFC 8785.

---

## 6. Middleware Airlock

Orchestrators support pluggable middleware chains intercepting data at trust boundaries:
1. `on_outbound_task`: Invoked prior to queue insertion (prompt injection filtering, PII masking).
2. `on_inbound_result`: Invoked upon result receipt (toxicity detection, data demasking).
3. `on_inbound_event`: Invoked upon event receipt (rate limiting, signal filtering).

Middleware exceptions **MUST** isolate and quarantine the affected job without crashing the orchestrator engine. Each middleware call **MUST** be bounded by a timeout (default: 5.0s).

---

## 7. Extended Specifications & Architecture

The normative standard comprises the core protocol specifications and specialized component chapters:

| Document | Scope | Status |
|---|---|---|
| [RXON Protocol](rxon-v1.1.md) | Handshake version negotiation (426 Upgrade Required), skill namespacing (`namespace/name`), SemVer 2.0.0 matching with build metadata, schema covariance/contravariance, `BlobReference` CAS offloading. | Normative Standard |
| [HoloReg CAS & Merkle DAG Registry](registry-cas.md) | OCI v1.1 Content-Addressable Storage (`blobs/sha256/`), relational Merkle DAG index, upload sessions, SemVer Gate, Referrers API, cluster leases & GC. | Normative Standard |
| [HLN Manifest & Bundling](manifest-and-bundling.md) | Project manifest `hln.toml`, lockfile `hln.lock`, `CanonicalManifest`, and deterministic `.tar.zst` bundling. | Normative Standard |
| [HoloNode Worker Runtime](worker-runtime.md) | Dynamic skill mounting via `mounter.py`, `uv venv` sandbox environments, IPC via Unix Domain Sockets (`/tmp/hln-ipc-*.sock`), pull-based skill synchronization via Heartbeat, SLSA/SBOM verification. | Normative Standard |
| [Holarchy Orchestrator Topology](orchestrator-topology.md) | `TopologyManager`, compiled `graph.rxon.json` execution without raw Python evaluation on Control Plane, worker groups `[topology.groups]`, federated `[topology.external]` holons. | Normative Standard |
| [Blueprint & FSM Execution](blueprint.md) | Declarative state machine DSL, state transitions, deciders, conditional routing, and sub-blueprint invocation. | Normative Standard |
| [Canonical JSON (JCS)](canonical-json.md) | RFC 8785 byte canonicalization for deterministic signatures and Merkle hashing. | Normative Standard |
| [Lifecycle & Status Transitions](lifecycle-and-status.md) | Finite State Machine transitions for Jobs, Tasks, and Workers. | Normative Standard |
| [Observability Protocol](observability-protocol.md) | W3C Distributed Tracing, OTLP telemetry metrics, and D-DNA audit logging. | Normative Standard |
| [Conformance Profiles](conformance.md) | Strict conformance verification criteria for Profile `L1-Baseline` and Profile `L2-Hardened`. | Normative Standard |
