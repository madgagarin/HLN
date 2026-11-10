# Holarchy: Orchestrator Topology & Control Plane Specification

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Conformance:** RFC 2119 / RFC 8174  

---

## 1. Architectural Scope

`holarchy` acts as the deterministic Control Plane for distributed holonic execution. It coordinates FSM state machines, schedules tasks across heterogeneous worker pools, enforces cryptographic and policy boundaries, and manages deployment topologies.

---

## 2. Declarative Topology Management (`TopologyManager`)

To maintain strict separation of concerns and eliminate remote code execution risks on the Control Plane:

### 2.1. Compilation vs Execution
* **Untrusted Code Isolation:** Control plane servers NEVER execute raw Python code from user blueprints or skills.
* **Compiled Declarative Graph (`graph.rxon.json`):** Blueprints are compiled ahead-of-time (via `hln push` or `hln deploy`) into a declarative JSON graph containing nodes, transitions, matching predicates, and topology configurations.
* **Loading:** `TopologyManager` reads the verified `graph.rxon.json` directly from CAS without evaluating dynamic Python code.

### 2.2. Topology Groups (`[topology.groups]`)
Blueprints declare internal execution requirements:
```toml
[topology.groups]
parsers = { skills = ["vision/ocr-fast@^1.4"], replicas = 2, min_vram_mb = 0 }
heavy_ai = { skills = ["llm/summarizer@^2.0"], replicas = 1, min_vram_mb = 16384 }
```
`TopologyManager`:
1. Resolves required skill versions via `holoreg`.
2. Records target `desired_skill_digests` for matching `holonode` instances in state storage, which are reconciled via `mount_digests` and `unmount_digests` directives in the Heartbeat response (pull-based synchronization).
3. Monitors replica counts and auto-heals failed sub-workers.

### 2.3. External Federated Skills (`[topology.external]`)
```toml
[topology.external]
verifier = { skill = "corp/kyc@^2.0", min_online_providers = 1, max_latency_ms = 3000 }
```
Allows cross-organizational delegation to peer holons:
* Bounded by latency targets (`max_latency_ms`).
* Authenticated via mTLS or STS tokens.
* Enforces provenance chain recording across organizational boundaries.

### 2.4. Leases & Heartbeat Management
* The orchestrator periodically reports active workload manifest digests to `holoreg` via `POST /v1/leases/heartbeat`.
* Protects in-flight and scheduled tasks from being purged by background CAS garbage collection.

---

## 3. Multi-Candidate Version Dispatching (`Dispatcher`)

The dispatcher matches incoming tasks to available worker nodes:

1. **Full Candidate Evaluation:** The dispatcher evaluates ALL skill candidates registered by a worker without premature `break` on name matches, allowing a single node to host multiple versions (e.g. `1.3.0` and `1.4.2`).
2. **SemVer Matching:** Evaluates requirements via `SkillInfo.matches()`:
   * Strips build metadata unless explicit lock pin is present.
   * Directs traffic to optimal candidates conforming to policy constraints.
3. **Capacity & Priority Filtering:** Separates match filtering (capacity, policy, tags) from prioritization (reputation, latency, cost).
