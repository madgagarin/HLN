# HLN Security Specification & Threat Model

This document specifies the security architecture, threat model, coordinated vulnerability disclosure policy, and formal mappings for the **Holarchical Logic Network (HLN)** protocol.

---

## 1. Coordinated Vulnerability Disclosure (CVD)

The HLN project is committed to maintaining protocol security and protecting distributed deployments. We follow a coordinated vulnerability disclosure process.

### 1.1. Reporting a Vulnerability

If you identify a security vulnerability in the HLN specification, reference schemas, or reference implementations (`holarchy`, `holonode`, `rxon`, `holoreg`, `holarchy-cli`), please do not open a public issue. Report it privately through one of the following channels:

*   **GitHub Security Advisories (Preferred):** Submit a private report via [GitHub Private Vulnerability Reporting](https://github.com/madgagarin/HLN/security/advisories/new).
*   **Maintainer Contact:** `madgagarin@gmail.com` (Dmitry Gagarin).

### 1.2. Response Service Level Agreements (SLA)

The HLN Security Working Group commits to the following timelines:

| Stage | SLA Window | Description |
| :--- | :--- | :--- |
| **Initial Acknowledgment** | $\le 24$ hours | Confirmation of receipt with an assigned vulnerability handler. |
| **Triage & Severity Assessment** | $\le 72$ hours | Validation of finding, STRIDE mapping, and CVSS v3.1/v4.0 scoring. |
| **Mitigation & Fix Delivery** | $\le 30$ calendar days | Development of normative spec amendment, test vectors, and patches. |
| **Public Disclosure & Advisory** | Coordinated | Release of CVE advisory following patch deployment window. |

### 1.3. Safe Harbor Statement

HLN considers security research conducted in good faith under this policy to be authorized. We commit to:
1.  Not pursuing legal action against researchers who discover and report vulnerabilities in accordance with this policy.
2.  Working collaboratively with researchers to validate and resolve reported issues before public disclosure.
3.  Crediting researchers in public advisories, changelogs, and release notes (unless anonymity is requested).

---

## 2. Architecture & Data Flow Diagram (DFD)

The HLN protocol divides distributed multi-agent systems into four distinct planes separated by strict trust boundaries.

```mermaid
flowchart TD
    subgraph ClientPlane["Client Plane (Untrusted / External)"]
        Client["Client / API Consumer"]
    end

    subgraph TrustBoundary1["--- Trust Boundary: Tenant Ingress (TLS 1.3 + Token Auth) ---"]
    end

    subgraph ControlPlane["Control Plane (Orchestrator PDP / PEP)"]
        Ingress["API Ingress & Rate Limiter<br/>(429 / 503 + Retry-After)"]
        FSM["Blueprint FSM & Decider Engine<br/>(HTN Decomposition)"]
        Dispatch["Task Dispatcher & Matcher<br/>(Monotonic Policy Attenuation)"]
        Watch["Watcher & Lease Monitor<br/>(fencing: job_epoch / attempt)"]
    end

    subgraph TrustBoundary2["--- Trust Boundary: Storage Isolation ---"]
    end

    subgraph StoragePlane["Storage Plane (Secured Infrastructure)"]
        Redis["Job State Storage (Redis)<br/>• HMAC State Signature<br/>• Nonce Deduplication Cache<br/>• Leases & Watch Keys"]
        S3History["History & Audit Storage (S3/DB)<br/>• Keyed D-DNA Audit Chains<br/>• Evidence Cards"]
    end

    subgraph TrustBoundary3["--- Trust Boundary: Worker Egress/Ingress (Reverse Pull Only) ---"]
    end

    subgraph WorkerPlane["Worker Plane (Distributed Execution Nodes)"]
        Shell["Worker Shell (PEP)<br/>• Zero Inbound Open Ports<br/>• Task Envelope Verification<br/>• Heartbeat / Cooperative Cancellation"]
        Ghost["Worker Ghost (Probabilistic Agent / Tool)<br/>• Sandboxed Execution<br/>• Contract Schema Output"]
    end

    Client -->|HTTPS POST /jobs| Ingress
    Ingress --> FSM
    FSM -->|Atomic State Update| Redis
    FSM --> Dispatch
    Dispatch -->|Enqueue Task| Redis
    Watch -->|Lease Expiry / Quarantine| FSM

    Shell -->|Long-Poll GET /claim POST| Ingress
    Shell -->|Append-Only provenance_chain| Ingress
    Shell -->|PATCH /heartbeat<br/>(cancel_requested)| Ingress
    Shell --> Ghost
    Ghost -->|Contract Output| Shell
    Shell -->|TaskResult (strip policy)| Ingress

    FSM --> S3History
```

---

## 3. STRIDE Threat Model

The following matrix documents the STRIDE threat analysis for the HLN protocol and defines normative mitigations across all components.

| STRIDE Category | Target Protocol Component | Threat Scenario | Protocol Impact | Normative HLN Mitigation | Conformance Profile |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Spoofing** | Worker Registration & Task Claim | Rogue node registers using forged identity or claims tasks belonging to another worker. | Task interception, rogue code execution. | Mandatory token validation (L1) or mutual TLS (mTLS) with x509 SAN identity matching (L2); audience-bound envelope signing (`target_worker_id`). | Profile L1 / L2 |
| **Spoofing** | Orchestrator Ingress | Attacker spoofs Orchestrator to inject rogue commands to workers. | Malicious task execution on worker machine. | Worker initiates connection (Reverse Axon, no inbound ports); validates orchestrator certificate against pinned CA; verifies task envelope signature. | Profile L1 / L2 |
| **Tampering** | Task Envelope in Transit | Adversary modifies task params, deadline, policy, or target skill in transit. | Privilege escalation, arbitrary tool invocation. | Full envelope signing covering 15 fields via RFC 8785 Canonical JSON (JCS); verified by worker before execution. | Profile L1 / L2 |
| **Tampering** | Event Bubbling | Malicious intermediary modifies or reorders bubbling event payloads. | Corrupted state transitions in parent holons. | Append-only `provenance_chain` with nested hop signatures: each hop binds `payload_hash`, `prev_sig`, `hop_id`, and `ts`. | Profile L1 / L2 |
| **Tampering** | Orchestrator Job State | Adversary tampers with raw serialized state in Redis. | State injection, skipped transitions. | Keyed HMAC-SHA256 signature (`_signature`) over all state fields (`state_version`, `transition_count`, `depth`); verification failure triggers quarantine. | Profile L1 / L2 |
| **Tampering** | Execution Audit Trail | Operator or rogue process alters historical audit logs to conceal misuse. | Audit failure, unprovable compliance. | Keyed HMAC-SHA256 D-DNA hash chain with strict monotonic sequence numbers (`seq_no`); tamper-evident under Profile L2. | Profile L1 / L2 |
| **Repudiation** | Multi-hop Event Origin | Sub-worker denies having dispatched a faulty or unauthorized event. | Blame misattribution in compound holons. | Cryptographic `provenance_chain` records immutable signature of origin worker and each intermediate relayer. Profile L2 enforces Ed25519 non-repudiation. | Profile L1 / L2 |
| **Information Disclosure** | Data in Transit | Sniffing network packets between orchestrators and workers. | Leakage of confidential input params or results. | Mandatory TLS 1.3 transport encryption; ephemeral STS tokens; envelope encryption for large payloads. | Profile L1 / L2 |
| **Information Disclosure** | Storage Zone Leakage | Unauthorized worker accesses global job state or other tenant context. | Cross-tenant data leakage. | Partitioned storage zones (`input`, `steps`, `metadata`); workers only receive task-scoped parameter slices, never raw global state. | Profile L1 / L2 |
| **Denial of Service** | Orchestrator Ingress Flooding | Malicious worker or client floods orchestrator with requests. | Orchestrator starvation, resource exhaustion. | Per-worker and per-credential token-bucket rate limiting returning HTTP `429` with mandatory `Retry-After` header. | Profile L1 / L2 |
| **Denial of Service** | Worker Open Ports Exploitation | External attacker launches port scans or exploits against worker hosts. | Worker compromise. | Architectural pull model: workers have **zero open inbound ports**; all communication is outbound long-polling or WebSocket. | Profile L1 / L2 |
| **Denial of Service** | Infinite Holarchy Recursion | Malicious Blueprint creates unbounded nested sub-blueprints. | Compute exhaustion, infinite loop. | Protocol bounds depth via `max_holarchy_depth` (default: 10); bounds loop executions via `max_transitions` / `transition_count`. | Profile L1 / L2 |
| **Denial of Service** | Split-Brain Stale Execution | Severed network worker continues processing expired task; submits result after retry. | Double execution, corrupted state race condition. | Fencing tokens `(job_epoch, attempt)` + lease TTL; orchestrator rejects stale epoch/attempt submissions with HTTP `409 Conflict`. | Profile L1 / L2 |
| **Elevation of Privilege** | Policy Escalation | Sub-blueprint or child task attempts to grant itself more skills than parent. | Unauthorized skill or resource access. | **Monotonic Policy Attenuation**: child policy can only narrow permissions ($\text{allowed\_skills}_{\text{child}} \subseteq \text{allowed\_skills}_{\text{parent}}$). | Profile L1 / L2 |
| **Elevation of Privilege** | Worker Policy Overwrite | Worker attempts to return modified policy dictionary in `TaskResult`. | Privilege escalation in parent state. | Orchestrator explicitly ignores and strips any `policy` key returned in `TaskResult`. | Profile L1 / L2 |
| **Elevation of Privilege** | Budget Overrun | Compound holon distributes full parent budget to every parallel branch. | Unbounded cost multiplication ($N \times \text{budget}$). | Divisible budget reservation: parallel branches divide parent `budget_cap` ($B / N$); sum of child budgets cannot exceed parent remaining budget. | Profile L1 / L2 |

---

## 4. OWASP Top 10 for LLM Applications (2025/2026) Mapping

The HLN architecture natively addresses the OWASP Top 10 for Large Language Model Applications:

| OWASP LLM Ref | Vulnerability Name | Protocol Risk in Distributed AI | HLN Architectural Countermeasure |
| :--- | :--- | :--- | :--- |
| **LLM01** | **Prompt Injection** | Adversarial input hijacks agent execution flow to execute arbitrary workflows. | **Deterministic FSM Control Plane**: Natural language prompts cannot alter the state machine topology. Transitions and allowed next states are strictly defined in declarative Blueprints. |
| **LLM02** | **Insecure Output Handling** | Unvalidated model output executed as system commands or written directly into state. | **Contract-Driven Schema Validation**: `TaskResult.data` is validated against the skill's declared `output_schema` before merging into `job_state.steps`. Schema violation triggers contract penalties. |
| **LLM03** | **Training Data Poisoning** | Malicious fine-tuning or poisoned RAG corpus corrupts agent behaviors. | **Artifact Integrity Hashing**: All installed artifacts and models are pinned by cryptographic digests (`InstalledArtifact.hash`). Skills declare strict risk tiers and capability boundaries. |
| **LLM04** | **Model Denial of Service** | Resource-intensive or recursive prompt payloads exhaust compute. | **Leases & Clamped Deadlines**: Every task execution is bounded by `lease_ttl_s`, `dispatch_timeout`, and `result_timeout`. Child timeouts are strictly clamped to parent remaining time. |
| **LLM05** | **Supply Chain Vulnerabilities** | Compromised third-party skills, dependencies, or base models. | **Skill Hashing & Registration**: Workers publish cryptographic `skills_hash` at registration. Any modification requires full resynchronization. Dual-licensing and verified provenance chains. |
| **LLM06** | **Sensitive Information Disclosure** | Accidental leakage of PII or proprietary parameters across holon layers. | **Monotonic Attenuation & Storage Zones**: Child holons only receive explicit sub-blueprint `input` parameters; parent storage zones are isolated. Trace context redaction prevents header leaks. |
| **LLM07** | **Insecure Plugin Design** | Agent invokes external tools with uncontrolled permissions or invalid parameters. | **Policy Enforcement Points (PEP)**: Skills declare input/output schemas and required permissions. Monotonic policy attenuation prevents unauthorized skill invocation. |
| **LLM08** | **Excessive Agency** | Agent takes irreversible real-world actions without human oversight. | **Human-in-the-Loop State**: Blueprints support `waiting_for_human` approval states with one-time approval tokens. Deciders only choose among predefined candidate states. |
| **LLM09** | **Overreliance** | Systems trust unverified probabilistic LLM output for critical decisions. | **Evidence Cards & Audit Replay**: Complete execution decisions generate cryptographic receipts and Markdown evidence cards. D-DNA hash audit chains allow forensic verification. |
| **LLM10** | **Model Theft** | Adversary extracts weights or proprietary logic through unauthorized API queries. | **Zero Trust Authentication**: Ephemeral STS tokens, mTLS node verification, and no exposed worker ports prevent unauthorized model interrogation. |

---

## 5. Security Conformance Profiles

HLN defines two formal conformance profiles detailed in [spec/conformance.md](spec/conformance.md):

*   **Profile L1 (Baseline):** Mandatory for all conforming implementations. Employs TLS 1.3, symmetric HMAC-SHA256 for envelope and job state signing, RFC 8785 JCS canonicalization, keyed D-DNA logging, and credit-based rate limiting with `Retry-After`.
*   **Profile L2 (Hardened):** Required for regulated environments (financial, healthcare, defense). Requires mTLS with x509 SAN identity matching, asymmetric Ed25519 (RFC 8032) signing for non-repudiation, STS token rotation, and hardware/PEP anchoring.
