# HLN Security Model: Zero Trust by Design

Traditional distributed systems often rely on open inbound ports or complex VPNs to allow the central server to push tasks to workers. This creates a massive attack surface.

HLN solves this via the **Reverse Command** pattern.

## 1. No Inbound Ports (The Pull Model)

HLN Workers act as **clients**, not servers. They initiate all connections to the Orchestrator.
*   **Zero Attack Surface:** A worker can run behind a NAT, inside a strict corporate firewall, or on a laptop in a coffee shop. No one can initiate a connection to a worker from the outside.
*   **Topology Agnostic:** The Orchestrator does not need to know the IP addresses of its workers. It simply places tasks in a queue.

## 2. mTLS & Token Rotation

HLN mandates a "Defense in Depth" approach for authentication.

### Level 1: mTLS (Mutual TLS)
Every component (Orchestrator and Worker) identifies itself via x509 certificates.
*   The Orchestrator verifies the Worker's certificate.
*   The Worker verifies the Orchestrator's certificate.
*   **Benefit:** Even if an attacker steals an auth token, they cannot connect without the corresponding private key/certificate.

### Level 2: STS (Security Token Service)
Certificates are used only to establish the secure channel. For API authorization, HLN uses short-lived **Access Tokens**.
*   Workers exchange their identity (Certificate) for an Access Token via the Orchestrator's built-in STS.
*   Tokens expire quickly (e.g., 1 hour).
*   **Benefit:** Automatic key rotation and granular permission scoping.

## 3. Protocol Integrity

HLN protocols include mechanisms for:
*   **Payload Verification:** All blob and large-payload transfers are verified via checksums (e.g., ETags) to prevent tampering in transit.
*   **Strict Typing:** The protocol enforces strict data schemas, rejecting malformed packets before they reach business logic.

---

## 4. Conceptual Security Framework

This document covers the *connectivity* security model — Pull, mTLS, STS, and payload integrity. For the **conceptual** security framework — which establishes the **TEE (Trusted Execution Environment)** via constrained autonomy (Policy), recursive protection (Holarchy Depth), cryptographic audit (D-DNA), Job State Signing, Outbound Task Signing, and middleware trust boundaries — see [Security Extensions](concepts/SECURITY_EXTENSIONS.md) and the [Specification](SPECIFICATION.md) (Sections 4.3–4.6 and 6).