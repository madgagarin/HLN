# Security Extensions: Policy, Depth & Cryptographic Audit

> [!NOTE]
> This document is informative (non-normative). In case of conflict, normative specifications in `spec/` take precedence.

> **"Trust no one. Verify everything. At every layer."**

This document describes the architectural foundations of the **HLN Security Extensions**. Together, these mechanisms transform the Orchestrator from a simple task planner into a **PEP (Policy Enforcement Point)** in accordance with NIST SP 800-207 for recursive AI agents.

---

## 1. The Problem: Autonomy Without Boundaries

In the original HLN architecture, a Holon (Ghost + Shell) is **autonomous** — it can execute any skill, call any sub-blueprint, and manage its own state. This autonomy is powerful, but it creates risks:

1. **Unbounded Recursion**: A blueprint can spawn a sub-job that re-enters the same blueprint, creating an infinite loop of nested jobs.
2. **Skill Hijacking**: A compromised worker could execute skills outside its intended scope.
3. **State Tampering**: An attacker with access to the shared state store could modify job states to redirect execution.
4. **Log Forgery**: Without cryptographic proof, execution logs can be retroactively altered.

**The Core Insight**: Autonomy without boundaries is chaos. We need **constrained autonomy** — freedom within a clearly defined perimeter.

---

## 2. The Three Pillars of HLN Security

### Pillar 1: Policy (Constrained Autonomy)

**Policy** is the "contract" that defines what a Holon is allowed to do. It's not about restricting creativity — it's about ensuring **predictability** in critical systems.

```python
policy = {
    "allowed_skills": ["transcribe", "summarize"],
    "budget_cap": 10.0
}
```

**Philosophy:**
- **Whitelists over Blacklists**: Explicitly define what's allowed, not what's forbidden.
- **Monotonic Attenuation**: Policies flow down the holarchy and can only be restricted further ($\text{allowed\_skills}_{\text{child}} \subseteq \text{allowed\_skills}_{\text{parent}}$).
- **Transparency**: Every task carries its policy, making violations immediately visible.

**Use Cases:**
- **Multi-tenant SaaS**: Each customer gets a policy limiting which skills they can use.
- **Cost Control**: `budget_cap` and divisible quota reservations prevent runaway spending in recursive orchestration.
- **Compliance**: Regulatory requirements enforced at the protocol level.

---

### Pillar 2: Holarchy Depth (Recursive Protection)

**Holarchy Depth** is the "speed limit" for recursive orchestration. It prevents the system from collapsing under unbounded complexity.

```
Layer 0: Root Job (depth = 0)
Layer 1: Sub-job (depth = 1)
Layer 2: Sub-sub-job (depth = 2)
...
Layer 10: BLOCKED (depth >= max_holarchy_depth, default: 10)
```

**Philosophy:**
- **Fractal ≠ Unbounded**: Fractals are self-similar, but unbounded recursion is a bug.
- **Predictable Resource Usage**: Depth limits ensure bounded resource consumption.
- **Debugging**: Deep holarchies are hard to debug. A depth limit keeps systems comprehensible.

**Independence from `transition_count`:**
- `transition_count` protects against **local loops** (A→B→A→B within one job).
- `holarchy_depth` protects against **recursive calls** (A→spawns B→spawns A across jobs).

They are **complementary**, not redundant.

---

### Pillar 3: Cryptographic Audit (D-DNA)

**D-DNA (Temporal Evidence Chaining)** provides cryptographic audit trails for execution logs.

```
Record 0: genesis_mac = HMAC-SHA256(K_audit, JCS({"job_id": id, "seq": 0, "prev_mac": "", ...}))
Record 1: record_mac_1 = HMAC-SHA256(K_audit, JCS({"job_id": id, "seq": 1, "prev_mac": genesis_mac, ...}))
Record 2: record_mac_2 = HMAC-SHA256(K_audit, JCS({"job_id": id, "seq": 2, "prev_mac": record_mac_1, ...}))
```

**Philosophy:**
- **Trust, but Verify**: We verify audit chains with keyed cryptographic signatures and canonical serialization (RFC 8785).
- **Tamper Evidence**: Once written, log records cannot be altered without cryptographic detection under Profile L2 with external anchoring (tamper-noticing under Profile L1).
- **Forensics**: When things go wrong, D-DNA identifies the exact event index where integrity was compromised.

**Lightweight Integrity:**
- D-DNA is **job-scoped**, avoiding global consensus overhead.
- Leverages RFC 8785 (JCS) and keyed HMAC-SHA256 or asymmetric Ed25519 signatures.

---

## 3. Job State Signing: Protecting the Control Plane

The **job state** holds the current state, transition counter, and policy. If an attacker can modify it in the shared state store, they control execution flow.

**Solution**: Sign the canonical JSON representation of the job state with the cluster secret key:

```python
signed_fields = {
    "id": job_state["id"],
    "state_version": job_state["state_version"],
    "current_state": job_state["current_state"],
    "transition_count": job_state["transition_count"],
    "holarchy_depth": job_state["holarchy_depth"],
    "policy": job_state["policy"],
    "budget_spent": job_state["budget_spent"],
    "d_dna_head_hash": job_state["d_dna_head_hash"]
}
signature = HMAC-SHA256(cluster_secret_key, rfc8785_canonical_json(signed_fields))
```

**Protection:**
- **State-Store Tampering**: If someone modifies `policy.allowed_skills` in the shared store, verification fails.
- **Rollback Protection**: Linearizable Compare-And-Swap (CAS) on `state_version` prevents state reversion.

---

## 4. Outbound Task Signing: Protecting Worker Queues

Tasks carry commands from orchestrator to workers. Signing prevents task injection, skill spoofing, parameter tampering, and replay attacks.

**Solution**: Sign the complete canonical task envelope using the target worker's token (Profile L1) or orchestrator's Ed25519 private key (Profile L2):

```python
task_envelope = {
    "job_id": task["job_id"],
    "task_id": task["task_id"],
    "attempt": task["attempt"],
    "job_epoch": task["job_epoch"],
    "type": task["type"],
    "params": task["params"],
    "worker_id": target_worker_id,
    "deadline": task["deadline"],
    "priority": task["priority"],
    "policy": task["policy"],
    "holarchy_depth": task["holarchy_depth"],
    "transition_count": task["transition_count"],
    "created_at": task["created_at"],
    "envelope_ts": task["envelope_ts"],
    "nonce": task["nonce"]
}
signature = HMAC-SHA256(worker_token, rfc8785_canonical_json(task_envelope))
task["security"]["orchestrator_signature"] = signature
```

**Protection:**
- **Task Injection**: Unsigned tasks or tasks signed with invalid keys are rejected.
- **Parameter & Skill Tampering**: Any modification of `type`, `params`, or `deadline` invalidates the signature.
- **Audience Binding**: The signature explicitly binds `worker_id`, preventing cross-worker task theft.

---

## 5. Middleware: Trust Boundaries

**Middleware** defines the **trust boundaries** between orchestrator cores, workers, and external systems:

```python
class PIIMaskingMiddleware(BaseMiddleware):
    async def on_outbound_task(self, context, task_payload):
        """Remove PII before sending to external workers"""
        task_payload["params"] = mask_pii(task_payload["params"])
        return task_payload

    async def on_inbound_result(self, context, result_payload):
        """Validate results don't contain toxic content"""
        if is_toxic(result_payload["data"]):
            raise MiddlewareError("Toxic content detected")
        return result_payload
```

**Philosophy:**
- **Defense in Depth**: Multiple layers of validation at boundary crossing.
- **Isolation**: Middleware errors quarantine the affected job without crashing the orchestrator.
- **Bounded Latency**: Each middleware execution is bounded by a timeout (default: 5s).

---

## 6. Zero Trust and the PEP in a Fractal World

The HLN security architecture implements a **Policy Enforcement Point (PEP)** per NIST SP 800-207 within the Orchestrator:

| Layer | Trust Model | Verification Mechanism |
|---|---|---|
| **Workers** | Not trusted | Envelope signature verification, audience binding, fencing tokens |
| **State store** | Not trusted | State signing, CAS optimistic concurrency, rollback detection |
| **Logs** | Not trusted | D-DNA keyed HMAC chaining with sequence numbers |
| **Sub-orchestrators** | Constrained | Monotonic policy attenuation + `max_holarchy_depth` limit |
| **Middleware** | Semi-trusted | Per-hook timeout + exception quarantine isolation |

---

## 7. Design Principles

### 7.1. Fail-Safe Defaults
- **Policy**: An empty policy imposes no extra skill restrictions, but cannot widen parent policies.
- **Depth**: Default depth = 0 (root job); maximum allowed depth defaults to 10.
- **Crypto Audit**: Opt-in for Profile L1; mandatory for Profile L2.

### 7.2. Independent Toggles
Security features can be enabled incrementally according to the deployment profile:
- **Policy enforcement** — skill whitelists and depth limits.
- **Cryptographic audit** — state/task signing and D-DNA.

---

## 8. Related Documents

- **Technical Specification**: `spec/SPECIFICATION.md`
- **Security Model**: `SECURITY.md`
- **Conformance Profiles**: `spec/conformance.md`

---

## 9. Conclusion

The Security Extensions transform HLN from an unconstrained agent environment into an enterprise-ready, auditable system.

**Baseline (Unprotected Mesh):**
- ❌ No policy enforcement
- ❌ No recursion protection (vulnerable to unbounded recursion loops)
- ❌ No log integrity (logs easily altered retroactively)
- ❌ No task signing (vulnerable to queue injection and parameter manipulation)
- ❌ No job state anti-tamper verification
- ❌ Unconstrained sub-job fan-out

**With Security Extensions:**
- ✅ Zero Trust identity (Profile L1 Baseline / Profile L2 Hardened)
- ✅ RFC 8785 JCS canonical cryptographic signatures
- ✅ **Policy enforcement** (constrained autonomy & monotonic attenuation)
- ✅ **Holarchy depth protection** (recursive safety bounded by `max_holarchy_depth`)
- ✅ **D-DNA log integrity** (keyed tamper-evident cryptographic audit)
- ✅ **Job state signing & CAS** (anti-tamper and anti-rollback)
- ✅ **Full task envelope signing** (injection and parameter tampering protection)
- ✅ **Middleware trust boundaries** (airlock data sanitization)

**The Result**: An architecture that is not merely autonomous, but **responsibly autonomous**.
