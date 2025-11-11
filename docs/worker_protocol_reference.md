# Worker Protocol Reference (RXON-HTTP)

This document describes the JSON-level API and message format for communication between nodes in an HLN network. It is aimed at implementers: anyone building an Orchestrator listener or a Worker shell.

> The logical protocol (message shapes, matching, security) is defined in the [Specification](../SPECIFICATION.md). This page describes how those messages map onto HTTP/WS endpoints in the default transport.
>
> **Path convention.** HLN defines the *shape* of these endpoints, not a mandated literal path. The paths below are expressed relative to a worker API root, shown here as `/_worker`; each implementation chooses its own root prefix.

## 1. Worker Registration

- **Endpoint:** `POST /_worker/workers/register`
- **Description:** The Worker announces its characteristics, resources, and skills to the Orchestrator.
- **Request body (JSON):**
  ```json
  {
    "worker_id": "my-worker-1",
    "worker_type": "gpu-worker",
    "supported_skills": [
      {
        "name": "transcribe",
        "schema_dialect": "json-schema",
        "properties": { "cost_per_token": 0.0001 }
      }
    ],
    "resources": {
      "max_concurrent_tasks": 5,
      "cpu_cores": 4,
      "ram_gb": 16.0,
      "devices": [
        {
          "type": "gpu",
          "model": "NVIDIA RTX 4090",
          "properties": { "memory_gb": 24 }
        }
      ],
      "properties": { "region": "us-east-1" }
    },
    "installed_software": { "python": "3.12" },
    "capabilities": {
      "hostname": "node-01",
      "cost_per_skill": { "transcribe": 1.0 }
    },
    "metadata": { "environment": "production" }
  }
  ```

## 2. Heartbeat

- **Endpoint:** `PATCH /_worker/workers/{worker_id}`
- **Request body (JSON):**
    ```json
    {
      "worker_id": "my-worker-1",
      "status": "idle",
      "usage": {
        "cpu_load_percent": 15.0,
        "ram_used_gb": 2.1,
        "devices_usage": [
          { "unit_id": "0", "load_percent": 0.0, "metrics": { "temp": 45 } }
        ]
      },
      "current_tasks": [],
      "available_skills": ["transcribe"],
      "hot_skills": ["whisper-v3"],
      "timestamp": 1717070000,
      "metadata": { "uptime": 3600 }
    }
    ```

## 3. Polling for a Task

- **Endpoint:** `GET /_worker/workers/{worker_id}/tasks/next`
- **Success response (200 OK):**
  ```json
  {
    "job_id": "uuid-job-123",
    "task_id": "uuid-task-456",
    "type": "transcribe",
    "params": { "audio_url": "blob://recordings/meeting.mp3" },
    "policy": {
      "allowed_skills": ["transcribe", "summarize"],
      "budget_cap": 10.0
    },
    "holarchy_depth": 2,
    "transition_count": 5,
    "security": {
      "signature": "...",
      "signer_id": "apex-orchestrator",
      "identity_chain": ["apex", "regional-proxy"],
      "orchestrator_signature": "hmac-sha256-signed-by-worker-token"
    },
    "timestamp": 1717070001,
    "metadata": { "priority_class": "high" }
  }
  ```

  Fields carried by a task:

  - `policy` — Execution policy: `allowed_skills` (skill whitelist) and `budget_cap` (optional cost limit).
  - `holarchy_depth` — Depth of this job in the Holarchy (0 for a root job).
  - `transition_count` — State-machine transition counter for the originating job.
  - `security.orchestrator_signature` — HMAC signature over the task, produced with the worker's own token, to prevent task injection.

## 4. Submitting a Result

- **Endpoint:** `POST /_worker/tasks/result`
- **Request body (JSON):**
  ```json
  {
    "job_id": "uuid-job-123",
    "task_id": "uuid-task-456",
    "worker_id": "my-worker-1",
    "status": "success",
    "data": { "text": "Hello world" },
    "policy": {
      "allowed_skills": ["transcribe", "summarize"],
      "budget_cap": 10.0
    },
    "holarchy_depth": 2,
    "transition_count": 5,
    "costs": { "tokens": 1420, "compute_seconds": 3.5 },
    "security": {
      "signature": "...",
      "identity_chain": ["apex", "regional-proxy", "my-worker-1"]
    },
    "timestamp": 1717070100
  }
  ```

  - `costs` — Expense metrics reported by the worker (tokens, compute seconds, etc.), used for budget tracking across the Holarchy.

## 5. Events (Worker-Initiated Signals)

Used for progress, logs, and alerts. See [Reactive HLN](../concepts/REACTIVE_HLN.md) for the conceptual model.

- **Endpoint:** `POST /_worker/events` (or over WebSocket)
- **Format:**
  ```json
  {
    "event_id": "uuid-event-789",
    "worker_id": "my-worker-1",
    "origin_worker_id": "my-worker-1",
    "event_type": "progress",
    "payload": { "progress": 0.5, "message": "Still working..." },
    "bubbling_chain": ["my-worker-1"],
    "timestamp": 1717070050,
    "security": { "signer_id": "my-worker-1" }
  }
  ```

## 6. Distributed Tracing (Observability)

The protocol carries OpenTelemetry context end to end through `tracing_context` (in tasks) and `trace_context` (in events and results).

- **Injection**: the Orchestrator passes `traceparent` inside `TaskPayload`.
- **Extraction**: the Worker uses that ID to create child spans for its execution.
- **Feedback**: the Worker returns the current context in results and events, keeping the trace graph intact across every layer of the Holarchy.

---
*This reference describes the default HTTP transport mapping of the RXON protocol.*
