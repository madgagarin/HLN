# HLN Protocol Glossary & Normative Terminology

This document establishes the official glossary of terms, normative definitions, and RFC conformance keywords for the **Holarchical Logic Network (HLN)** specification.

---

## 1. Conformance Keywords (RFC 2119 / RFC 8174)

The key words **"MUST"**, **"MUST NOT"**, **"REQUIRED"**, **"SHALL"**, **"SHALL NOT"**, **"SHOULD"**, **"SHOULD NOT"**, **"RECOMMENDED"**, **"NOT RECOMMENDED"**, **"MAY"**, and **"OPTIONAL"** in this document and throughout normative HLN specification documents are to be interpreted as described in BCP 14 [RFC 2119] and [RFC 8174] when, and only when, they appear in all capitals.

*   **MUST / SHALL / REQUIRED:** Absolute requirement of the specification.
*   **MUST NOT / SHALL NOT:** Absolute prohibition of the specification.
*   **SHOULD / RECOMMENDED:** Valid reasons may exist in particular circumstances to ignore a particular item, but the full implications must be understood and carefully weighed before choosing a different course.
*   **SHOULD NOT / NOT RECOMMENDED:** Valid reasons may exist when the particular behavior is acceptable or even useful, but the full implications should be understood and the case carefully weighed before implementing any behavior described with this label.
*   **MAY / OPTIONAL:** Truly optional; implementations that do not include the option MUST be prepared to interoperate with implementations that do.

---

## 2. Core Architectural Terminology

### Holon
*(Etymology: Arthur Koestler, «The Ghost in the Machine», 1967; from Greek ὅλον holon, neuter of ὅλος holos "whole".)*
A fundamental entity that is simultaneously an autonomous whole in relation to its sub-parts and a subordinate part in relation to a larger system. In HLN, a Holon can represent a composite state machine (Compound Holon), an atomic task executor (Atomic Holon), an AI agent, a physical hardware resource, or a human supervisor.

### Holarchy
A hierarchical organization of self-regulating, autonomous holons that cooperate to achieve systemic goals. Unlike strict authoritarian hierarchies, a holarchy preserves local autonomy and self-governance at lower tiers while ensuring goal alignment through bounded policy constraints.

### Apex Holon
The root coordinator or top-level orchestrator in a given holarchic execution context. The Apex Holon maintains the root execution state, initializes tenant quotas, and anchors the final deterministic audit record.

### SOHO (Self-Organizing Holarchic Open-system)
An open, dynamic holarchy capable of dynamically reconfiguring worker assignments, load routing, and failover topologies in response to network fluctuations, worker churn, and changing execution costs.

### Janus Effect
*(Named after the two-faced Roman god Janus.)*
The dual operational orientation of every holon:
1.  **Downwards-facing:** Displays autonomy, self-direction, and local control over internal execution branches.
2.  **Upwards-facing:** Displays integrative, cooperative obedience to parent policies, budget bounds, and cancellation directives.

### Ghost-in-the-Shell
The architectural pattern separating domain intelligence from protocol mechanics:
*   **Ghost:** The non-deterministic, probabilistic execution logic (e.g. LLM prompt runner, heuristic optimizer, robotic controller, human worker).
*   **Shell:** The deterministic, RFC-compliant adapter and Policy Enforcement Point (PEP) providing mTLS, envelope signing, schema validation, rate-limiting, and heartbeat telemetry.

---

## 3. Execution & Protocol Terminology

### Blueprint
The formal declarative specification (YAML/JSON) defining a holon's finite state machine (FSM), including states, transitions, timeouts, required skills, decider candidate lists, and parallel DAG topologies. Conforms to [schemas/blueprint.schema.json](schemas/blueprint.schema.json).

### StepScope
The variable isolation and visibility matrix governing parameter sharing across Blueprint states:
*   `self`: Step output is strictly private to the executing step and not shared.
*   `job`: Step output is merged into global job steps and accessible to downstream steps.
*   `global`: Step output is elevated to multi-tenant or parent holarchy scope (requires explicit policy authorization).

### Decider
An advisory, non-deterministic routing component within a Blueprint (such as an LLM classifier, rule engine, or human decider) that inspects current context and selects the next state strictly from a declared list of `candidates`. If the decider fails or times out, execution deterministically transfers to the designated `fallback` state.

### Fencing Token
A monotonically increasing tuple `(job_epoch, attempt)` assigned by the orchestrator upon task dispatch or claim. Fencing tokens prevent split-brain execution races: any task result submitted with a stale epoch or attempt is rejected with HTTP `409 Conflict`.

### Lease
A time-bounded reservation (`lease_ttl_s`) granted to a worker upon claiming a task. The worker must actively renew the lease by reporting the in-flight task ID in periodic heartbeat requests. If unrenewed before expiry, the orchestrator reclaims and re-queues the task.

### Bounded-Latency Cooperative Cancellation
The protocol directive by which a cancel request initiated at a parent holon propagates downward via heartbeat response payloads (`cancel_requested`). Upon receipt, worker Shells gracefully terminate the executing Ghost and report a final status of `cancelled`.

### Orphan Policy
The formal recovery behavior executed by a worker Shell upon detecting lost connectivity or expired orchestrator sessions:
*   `abort`: Immediately cancel in-flight tasks and release reserved resources.
*   `suspend`: Pause task execution and wait up to a bounded reconnection timeout.
*   `continue_bounded`: Continue processing up to the remaining duration of `lease_ttl_s`, aborting if the lease expires before connection is restored.

### Monotonic Policy Attenuation
The invariant rule that a child holon or sub-blueprint can only narrow, never expand, the execution permissions and resources granted by its parent:
$$\text{allowed\_skills}_{\text{child}} \subseteq \text{allowed\_skills}_{\text{parent}}$$

### Divisible Budget
The resource governance constraint ensuring that parent `budget_cap` allocations are partitioned across child holons or parallel branches ($B / N$), preventing unbounded compute and cost multiplication across recursive holarchies.

### Event Bubbling
*(Derived from Stafford Beer's algedonic signal and Chain-of-Responsibility pattern.)*
The mechanism whereby an unhandled event or progress notification generated at a leaf holon ascends sequentially through parent holons until an ancestor containing an explicit handler for that `event_type` intercepts and processes it.

### Provenance Chain
An append-only, cryptographically linked series of nested signature hops (`provenance_chain`) embedded within bubbling events. Each hop binds the payload digest, previous hop signature, worker identity, and timestamp, guaranteeing non-repudiation and tamper-evidence across multi-tier holarchies.

### D-DNA (Deterministic Distributed Audit Trail)
A cryptographic ledger of state transitions and execution events, secured via keyed HMAC-SHA256 (Profile L1) or digital signatures (Profile L2) over canonical JSON representations with strict monotonic sequence numbers (`seq_no`).

### Evidence Card
A human-readable Markdown card paired with machine-readable cryptographic receipts that documents why a probabilistic Ghost made a specific execution choice, enabling forensic review and human-in-the-loop compliance.

---

## 4. Security & Conformance Terminology

### Policy Enforcement Point (PEP)
Per NIST SP 800-207 Zero Trust Architecture, the gatekeeper component (implemented in the Orchestrator Ingress and Worker Shell) that intercepts, verifies, and enforces authorization policies before allowing any request to proceed to the Ghost or internal storage.

### Profile L1 (Baseline)
The standard HLN conformance tier mandatory for all implementations. Specifies TLS 1.3, symmetric HMAC-SHA256 envelope signing, RFC 8785 Canonical JSON (JCS), keyed D-DNA logging, and token-bucket rate limiting with `Retry-After`.

### Profile L2 (Hardened)
The high-assurance HLN conformance tier for regulated enterprise and defense systems. Specifies mutual TLS (mTLS) with x509 SAN identity matching, asymmetric Ed25519 (RFC 8032) envelope signatures, STS token rotation, and hardware/PEP anchoring.
