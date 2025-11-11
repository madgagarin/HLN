# HLN: Holarchical Logic Network

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-brightgreen.svg)](https://creativecommons.org/licenses/by/4.0/)

**HLN (Holarchical Logic Network)** is an architectural pattern designed to orchestrate autonomous, distributed agents in a secure, predictable, and scalable manner.

Unlike traditional **A2A (Agent-to-Agent)** networks where behavior is emergent and often chaotic, HLN introduces a structured **Holarchy** (a hierarchy of autonomous units). It transforms unpredictable AI swarms into reliable, fractal business processes.

## Key Principles

### 1. The Holon (Ghost-in-the-Shell)
The fundamental unit of HLN is the **Holon**. Every Holon follows a simple pattern:
*   **The Ghost**: The internal logic (an AI model, a script, or a human expert).
*   **The Shell**: A standardized interface that makes the Ghost part of the network using the **RXON** protocol.

By combining them, we get a predictable building block that can be stacked like nesting dolls (**Matryoshka**).

### 2. RXON (Reverse Axon Protocol)
The "nervous system" of the HLN. Nodes in the network communicate using the [**RXON**](https://github.com/madgagarin/RXON) (Reverse Axon) protocol. The superior node (Ghost) **never** connects to the subordinate node (Shell). Instead, the Shell initiates the connection.

A **Reverse Axon** is a secure, worker-initiated tunnel:
*   **Bottom-Up Initiative**: The "muscle" (Shell) grows a nerve towards the "brain" (Ghost).
*   **No Open Ports**: Nodes run safely behind NAT, firewalls, or on mobile devices.
*   **Zero Trust**: Built-in support for **Identity Chains** and cryptographically verified message bubbling across infinite layers.

### 3. Shell-Stacked Holarchy (The Structure)
We build **Holarchies**, not simple Hierarchies.
*   **Fractal**: A complex cluster of Holons can be wrapped in a single Shell to appear as a simple specialist to a higher-level system.
*   **Recursive**: The pattern (Ghost -> Shell) repeats at every scale, allowing for infinite depth.

### 4. Smart Resource Matching
HLN treats hardware and expertise as abstract units of "Intent".
*   **Universal**: Any device (GPU, Sensor, Actuator) or human skill is matched by arbitrary `properties`.
*   **Smart Matching**: Intent is matched to reality using standardized **GE (Greater or Equal)** logic for numbers.

## Documentation

### Core Documents

*   [**The HLN Manifesto**](MANIFESTO.md): Why we need to tame the chaos of AI agents.
*   [**Specification**](SPECIFICATION.md): The technical details of the protocol.
*   [**Security Model**](SECURITY.md): Connectivity security, mTLS, and Zero Trust.
*   [**Worker Protocol Reference**](docs/worker_protocol_reference.md): JSON-level API description for implementers.

### Concepts & Deep Dives

*   [**Holarchy & Shell-Stacking**](concepts/HOLARCHY_MANIFESTO.md): Deep dive into the philosophical architecture.
*   [**Security Extensions**](concepts/SECURITY_EXTENSIONS.md): Policy enforcement, recursive protection, and cryptographic audit.
*   [**Reactive HLN**](concepts/REACTIVE_HLN.md): Worker-initiated events, signaling, and bubbling.
*   [**Distributed Tracing**](concepts/DISTRIBUTED_TRACING.md): How visibility is maintained across layers.
*   [**Pluggable Transports**](concepts/PLUGGABLE_TRANSPORTS.md): Decoupling the protocol from the physical network.
*   [**Physical Holons**](concepts/PHYSICAL_HOLONS.md): Extending the grid into the material world.

## Implementations

*   [**Avtomatika Engine**](https://github.com/avtomatika-ai/avtomatika): A production-ready Python implementation of the HLN Orchestrator.
*   [**Worker SDK**](https://github.com/avtomatika-ai/avtomatika-worker): Python SDK for building HLN-compliant workers.
*   [**RXON Protocol**](https://github.com/madgagarin/RXON): Lightweight reverse-connection (RXON) protocol for distributed systems. Supports universal resource telemetry (AI/Robotics/IoT) and strictly typed skill contracts. Works behind NAT/firewalls.


## Contributing

We welcome feedback and proposals! Please see [**CONTRIBUTING.md**](CONTRIBUTING.md) for guidelines on how to propose changes to the architecture.

## License

This specification is open-source and licensed under the **Creative Commons Attribution 4.0 International (CC BY 4.0)**.
