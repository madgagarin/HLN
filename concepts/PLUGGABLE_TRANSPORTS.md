# Pluggable Transport Layer

## Objective
To completely decouple the logical RXON protocol (Message Exchange) from the physical transport layer (Network). This allows the same Orchestrator logic to communicate with Workers via HTTP, WebSocket, Tor, gRPC, or even Carrier Pigeons, simply by switching a configuration driver.

---

## Architecture

### 1. The Core (the protocol core)
The protocol core MUST remain lightweight and dependency-free. It defines **Interfaces** (Abstract Base Classes) but contains **NO** network implementation code.

#### `Transport` (Interface for Workers)
Must be implemented by any transport adapter.
```python
class Transport(ABC):
    @abstractmethod
    async def connect(self) -> None:
        """Establish connection (handshake)"""
        pass

    @abstractmethod
    async def register(self, registration: WorkerRegistration) -> Any:
        """Announce capabilities and resources"""
        pass

    @abstractmethod
    async def poll_task(self, timeout: float = 30.0) -> TaskPayload | None:
        """Wait for a task (Pull)"""
        pass

    @abstractmethod
    async def send_result(self, result: TaskResult) -> bool:
        """Push a result"""
        pass
```

#### `Listener` (Interface for Orchestrators)
```python
class Listener(ABC):
    @abstractmethod
    async def start(self, handler: Callable[[str, Any, dict[str, Any]], Awaitable[Any]]) -> None:
        """Start listening for incoming worker connections"""
        pass
```

---

## 2. The Adapters (Plugins)

Implementations reside in submodules or separate packages. They introduce heavy dependencies (aiohttp, grpcio, stem) only if installed.

### `HTTP` (Default)
*   **Protocol:** HTTP/1.1 or HTTP/2
*   **Example library:** `aiohttp`
*   **Logic:** Implements Long-Polling for `poll_task`. Maps RXON messages to JSON POST requests.
*   **URL Scheme:** `http://`, `https://`

### `Onion` (Privacy)
*   **Protocol:** Tor Hidden Services
*   **Example library:** `stem` (Tor Controller)
*   **Logic:** Orchestrator creates a Hidden Service. Worker connects via SOCKS5 proxy through the Tor network.
*   **URL Scheme:** `onion://`
*   **Benefit:** Zero-configuration NAT traversal. No static IP needed for Orchestrator.

### `gRPC` (Performance)
*   **Protocol:** HTTP/2 + Protobuf
*   **Example library:** `grpcio`
*   **Logic:** Uses bi-directional streaming. Highly efficient for internal data center traffic.
*   **URL Scheme:** `grpc://`

---

## 3. The Factory (Integration)

A Worker SDK should not hardcode any specific HTTP client. Instead, it should use a Factory pattern.

```python
# Pseudo-code in a Worker SDK

# 1. Resolve the orchestrator endpoint from configuration
# (e.g., "http://...", "ws://...", or "onion://...")
orchestrator_url = resolve_orchestrator_endpoint()

# 2. Load the correct adapter dynamically based on the URL scheme
transport = create_transport(orchestrator_url)

# 3. Use the transport blindly
await transport.connect()
task = await transport.poll_task()
```

---

## Developer Guidelines (Do's and Don'ts)

### Don't
*   **Do NOT** import transport-specific libraries (`aiohttp`, `requests`, etc.) into the protocol core.
*   **Do NOT** put business logic (state transitions) inside transport adapters.
*   **Do NOT** assume the transport is synchronous or asynchronous (assume Async always).

### Do
*   **DO** ensure every transport adapter passes the standard transport compliance test suite.
*   **DO** handle transport-specific errors (e.g., Tor circuit failure) inside the adapter and raise a standard transport error to the core.
*   **DO** allow multiple listeners on the Orchestrator (e.g., listen on a public HTTP endpoint and a Tor Onion service simultaneously).
