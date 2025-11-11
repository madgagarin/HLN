# Security Extensions: Policy, Depth & Cryptographic Audit

> **"Trust no one. Verify everything. At every layer."**

This document describes the philosophical and architectural foundations of the **HLN Security Extensions**. Together, these mechanisms transform the Orchestrator from a simple task planner into a **TEE (Trusted Execution Environment)** for recursive AI agents.

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
- **Inheritance**: Policies flow down the holarchy unless explicitly overridden.
- **Transparency**: Every task carries its policy, making violations immediately visible.

**Use Cases:**
- **Multi-tenant SaaS**: Each customer gets a policy limiting which skills they can use.
- **Cost Control**: `budget_cap` prevents runaway spending in recursive orchestration.
- **Compliance**: Regulatory requirements (GDPR, HIPAA) enforced at the protocol level.

---

### Pillar 2: Holarchy Depth (Recursive Protection)

**Holarchy Depth** is the "speed limit" for recursive orchestration. It prevents the system from collapsing under its own complexity.

```
Layer 0: Root Job (depth = 0)
Layer 1: Sub-job (depth = 1)
Layer 2: Sub-sub-job (depth = 2)
...
Layer 10: BLOCKED (depth >= maximum depth)
```

**Philosophy:**
- **Fractal ≠ Infinite**: Fractals are beautiful, but infinite recursion is a bug.
- **Predictable Resource Usage**: Depth limits ensure bounded resource consumption.
- **Debugging**: Deep holarchies are hard to debug. A depth limit keeps systems comprehensible.

**Independence from `transition_count`:**
- `transition_count` protects against **local loops** (A→B→A→B within one job).
- `holarchy_depth` protects against **recursive calls** (A→spawns B→spawns A across jobs).

They are **complementary**, not redundant.

---

### Pillar 3: Cryptographic Audit (D-DNA)

**D-DNA (Temporal Evidence Chaining)** is the "black box" for execution logs. It provides **mathematical proof** that logs haven't been tampered with.

```
Event 1: genesis_hash = SHA-256(job_id + secret_salt)
Event 2: hash = SHA-256(Event1.hash + payload + timestamp)
Event 3: hash = SHA-256(Event2.hash + payload + timestamp)
```

**Philosophy:**
- **Trust, but Verify**: We trust our systems, but we verify with math.
- **Immutable History**: Once written, logs cannot be altered without detection.
- **Forensics**: When things go wrong, D-DNA tells you exactly where the chain broke.

**Why not blockchain?**
- D-DNA is **job-scoped**, not global. Each job has its own chain.
- No consensus required. No external infrastructure.
- Lightweight: Just SHA-256 hashes.

---

## 3. Job State Signing: Protecting the "Brain"

The **job state** is the "brain" of the orchestrator — it holds the current state, transition count, and policy. If an attacker can modify it, they control the execution flow.

**Solution**: Sign the job state with HMAC-SHA256 using a cluster-wide secret key.

```python
signed_fields = {
    "id": job_state["id"],
    "current_state": job_state["current_state"],
    "transition_count": job_state["transition_count"],
    "holarchy_depth": job_state["holarchy_depth"],
    "policy": job_state["policy"]
}
signature = HMAC-SHA256(cluster_secret_key, canonical_json(signed_fields))
```

**Protection:**
- **State-Store Tampering**: If someone modifies `policy.allowed_skills` in the shared state store, the signature won't match.
- **Cluster Consistency**: All orchestrator instances use the same key, ensuring consistent verification.

---

## 4. Outbound Task Signing: Protecting the "Nerves"

**Tasks** are the "nerves" of HLN — they carry commands from the brain (orchestrator) to the muscles (workers). If an attacker can inject fake tasks, they hijack the system.

**Solution**: Sign tasks with the worker's secret token before placing them in the queue.

```python
task_fields = {
    "job_id": task["job_id"],
    "task_id": task["task_id"],
    "transition_count": task["transition_count"],
    "holarchy_depth": task["holarchy_depth"],
    "policy": task["policy"]
}
signature = HMAC-SHA256(worker_token, canonical_json(task_fields))
task["security"]["orchestrator_signature"] = signature
```

**Worker Verification:**
```python
if not verify_signature(task, worker_token):
    raise SecurityError("Task signature invalid")
```

**Protection:**
- **Task Injection**: Fake tasks without valid signatures are rejected.
- **Replay Attacks**: Signatures include `task_id`, preventing replay of old tasks.

---

## 5. Middleware: Trust Boundaries

**Middleware** defines the **trust boundaries** between the orchestrator core, workers, and external systems. It's the "airlock" that sanitizes data as it crosses boundaries.

For example, a PII-masking middleware could hook the outbound and inbound boundaries (names below are illustrative):

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
- **Defense in Depth**: Multiple layers of validation, not just one.
- **Isolation**: Middleware errors quarantine the job, they don't crash the orchestrator.
- **Timeout**: Each middleware call has a timeout (default: 5s) to prevent slow middleware from blocking the system.

---

## 6. Zero Trust and the TEE in a Fractal World

The original HLN security model was **Zero Trust** — no node is trusted by default. The Security Extensions take this further by establishing a **Trusted Execution Environment (TEE)** within the Orchestrator itself:

| Layer | Trust Model | Verification |
|-------|-------------|--------------|
| **Workers** | Not trusted | Task signature verification |
| **State store** | Not trusted | Job state signing |
| **Logs** | Not trusted | D-DNA cryptographic chain |
| **Sub-orchestrators** | Constrained | Policy inheritance + depth limit |
| **Middleware** | Semi-trusted | Timeout + error isolation |

**The Mantra**: *"Trust no one. Verify everything. At every layer."*

---

## 7. Design Principles

### 7.1. Fail-Safe Defaults
- **Policy**: An empty policy imposes no restrictions.
- **Depth**: Default depth = 0 (root job).
- **Crypto Audit**: Disabled by default; opt in when the deployment is ready.

### 7.2. Independent Toggles
Every security feature can be enabled independently:
- **Policy enforcement** — skill whitelists and depth limits.
- **Cryptographic audit** — state/task signing and D-DNA.

This lets each concern be adopted independently, and any single concern suspended without affecting the rest.

### 7.3. Transparency
- All security fields are visible in API responses.
- Violations are logged with full context.
- Metrics exposed through the standard observability surface for monitoring.

---

## 8. Related Documents

- **HLN Specification**: `SPECIFICATION.md` (Sections 4.3-4.6)
- **Security Model**: `SECURITY.md`

---

## 9. Conclusion

The Security Extensions transform HLN from a **powerful but risky** architecture into a **predictable and auditable** system suitable for critical business processes.

**Baseline:**
-  Zero Trust identity
-  Cryptographic signatures
-  No policy enforcement
-  No recursion protection
-  No log integrity

**With Security Extensions:**
-  Zero Trust identity
-  Cryptographic signatures
-  **Policy enforcement** (constrained autonomy)
-  **Holarchy depth protection** (recursive safety)
-  **D-DNA log integrity** (cryptographic audit)
-  **Job state signing** (anti-tamper)
-  **Task signing** (injection protection)
-  **Middleware** (trust boundaries)

**The Result**: A system that's not just **autonomous**, but **responsibly autonomous**.

---

*This document is part of the HLN architectural specification.*
