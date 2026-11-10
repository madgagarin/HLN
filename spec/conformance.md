# HLN Conformance Profiles

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Conformance:** RFC 2119 / RFC 8174  

---

## 1. Scope & Objective

To eliminate ambiguity between security claims and implementation realities, HLN establishes two formal conformance profiles. Implementations **MUST** declare conformance to either **Profile L1-Baseline** or **Profile L2-Hardened**.

---

## 2. Profile Summary Matrix

| Capability / Requirement | Profile `L1-Baseline` (Trusted Grid) | Profile `L2-Hardened` (Zero-Trust Grid) |
|---|---|---|
| **Primary Environment** | Single-tenant private clusters, local development | Multi-tenant SaaS, cross-organizational grids, untrusted networks |
| **Transport Encryption** | Standard TLS 1.3 | Mutual TLS (mTLS) with client certificate CN verification |
| **Authentication** | Shared static worker tokens (`X-Worker-Token`) | Dynamic STS Access Tokens (`kid` rotation, short TTL $\le 1\text{h}$) |
| **Message Signing** | Symmetric `HMAC-SHA256` | Asymmetric `Ed25519` (RFC 8032) in JWS format (RFC 7515) |
| **Non-Repudiation** | Not provided (shared secret allows symmetric forging) | Cryptographically guaranteed via private key signatures |
| **Canonical Serialization** | RFC 8785 (JCS) | RFC 8785 (JCS) |
| **Audience Binding** | `worker_id` included in signed envelope | `worker_id` + certificate thumbprint bound in JWS payload |
| **D-DNA Audit Chaining** | Keyed `HMAC-SHA256` with sequence numbers | Keyed `HMAC-SHA256` + Periodic Signed Tree Head / TSA external anchor |
| **Bubbling Event Chain** | Chained HMAC (Macaroons pattern) | Append-only nested Ed25519 signatures in `provenance_chain` |
| **Integrity Qualification** | **Tamper-noticing** within shared trust boundary | **Tamper-evident** and non-repudiable across untrusted boundaries |

---

## 3. Profile L1: Baseline (Shared-Secret Trusted Grid)

Profile L1 is intended for rapid developer deployment and closed internal infrastructure:

1. **Transport Security:** Connections between Workers and Orchestrators **MUST** use TLS 1.2 or TLS 1.3.
2. **Authentication:** Workers identify themselves using static shared tokens via the `X-Worker-Token` header.
3. **Envelope Signing:** Outbound tasks and inbound results **MUST** be signed using `HMAC-SHA256` over the RFC 8785 canonical JSON representation of the envelope.
4. **Non-Repudiation Disclaimer:** Profile L1 **DOES NOT** provide non-repudiation because the shared secret is known to both Orchestrator and Worker. Tamper-evidence is bounded by the integrity of the shared cluster secret.

---

## 4. Profile L2: Hardened (Zero-Trust Grid)

Profile L2 enforces end-to-end zero-trust isolation:

1. **Mutual TLS (mTLS):**
   - The Orchestrator **MUST** demand and verify a valid client certificate on every TLS handshake.
   - The certificate Common Name (CN) or SAN URI **MUST** strictly match the registering `worker_id`. Handshake identity mismatches **MUST** terminate the connection.

2. **Security Token Service (STS) & Key Rotation:**
   - Workers exchange verified mTLS certificates for short-lived access tokens via `POST /_worker/auth/token`.
   - Tokens **MUST** expire within 3600 seconds (1 hour).
   - Tokens **MUST** include a key identifier (`kid`) to enable non-disruptive key rotation.

3. **Asymmetric Ed25519 Signing (RFC 8032):**
   - Every Worker and Orchestrator possesses an Ed25519 keypair.
   - Workers sign `TaskResult` and `WorkerEventPayload` using their private key.
   - Orchestrators verify signatures using the worker's registered public key.
   - Orchestrators sign `TaskPayload` with their cluster private key.
   - Provides true cryptographic non-repudiation: neither party can forge messages attributed to the other.

4. **Append-Only Nested `provenance_chain`:**
   - When an event bubbles through intermediate holons, each forwarding hop $i$ appends an entry:
     $$\text{sig}_i = \text{Sign}_{K_i}(\text{JCS}(\{\text{prev\_sig}: \text{sig}_{i-1},\ \text{hop\_id}: \text{id}_i,\ \text{ts}: t_i,\ \text{payload\_hash}: H(P)\}))$$
   - Any truncation, reordering, or alteration of intermediate hops is immediately detectable by verifying the public key chain.

5. **External Anchoring for D-DNA:**
   - Under Profile L2, D-DNA execution logs **SHOULD** publish periodic Merkle roots (Signed Tree Heads) to an external immutable ledger or RFC 3161 Time-Stamp Authority (TSA).

---

## 5. Conformance Verification & Test Vectors

Implementations claiming HLN conformance **MUST** pass the following validation suites:

1. **JCS Canonicalization Vectors:** Verify byte-level equality against `conformance/test-vectors/jcs.json` covering UTF-16 code unit key sorting, IEEE 754 number serialization, and empty structure normalizations.
2. **Task Envelope Signing Vectors:** Verify exact HMAC-SHA256 signature computation against `conformance/test-vectors/task_signing.json`.
3. **D-DNA Audit Sequence Vectors:** Verify sequential keyed HMAC chaining and tampering detection against `conformance/test-vectors/d_dna.json`.
4. **Machine-Readable Schema Validation:** Verify that all emitted payloads pass validation against the Draft 2020-12 schemas in `schemas/`.
5. **HTTP & Transport Wire Conformance:** Verify compliance with the endpoint contracts and status code semantics defined in `api/openapi.yaml`, including `204 No Content` on empty long-poll expiry, `409 Conflict` on stale fencing tokens `(job_epoch, attempt)`, `410 Gone` on expired or cancelled tasks, `426 Upgrade Required` on protocol version mismatch, and `Retry-After` adherence on `429` / `503` responses.
