# Worker Protocol Reference (RXON-HTTP Binding)

**Status:** Implementation Guide  
**Conformance:** [HLN Technical Specification](../spec/SPECIFICATION.md)  

This document describes the JSON-level HTTP API and wire format for communication between Orchestrators and Workers in an HLN network.

> **Path Convention:** Endpoints are expressed relative to the worker API root, default: `/_worker`.

---

## 1. Worker Registration

Announces the worker's skills, compute resources, software environment, and cryptographic identity.

- **Endpoint:** `POST /_worker/workers/register`
- **Headers:** `Content-Type: application/json`, `X-Worker-Token: <token>` (Profile L1)
- **Request Body (JSON):**
  ```json
  {
    "worker_id": "worker-gpu-node-01",
    "worker_type": "gpu-worker",
    "supported_skills": [
      {
        "name": "transcribe",
        "type": "audio-inference",
        "version": "1.2.0",
        "schema_dialect": "json-schema",
        "pricing": {
          "cost_per_token": 0.0001,
          "cost_per_task": 0.02,
          "currency": "USD"
        },
        "input_schema": {
          "type": "object",
          "required": ["audio_url"],
          "properties": {
            "audio_url": { "type": "string", "format": "uri" },
            "language": { "type": "string", "default": "en" }
          },
          "additionalProperties": false
        },
        "output_schema": {
          "type": "object",
          "required": ["text", "duration_s"],
          "properties": {
            "text": { "type": "string" },
            "duration_s": { "type": "number" }
          },
          "additionalProperties": false
        }
      }
    ],
    "resources": {
      "cpu_cores": 16,
      "ram_gb": 64.0,
      "devices": [
        {
          "type": "gpu",
          "model": "NVIDIA RTX 4090",
          "id": "gpu-0",
          "properties": {
            "vram_gb": 24.0,
            "compute_capability": "8.9"
          }
        }
      ],
      "properties": {
        "region": "us-east-1",
        "zone": "us-east-1a",
        "os": "linux",
        "arch": "x86_64"
      }
    },
    "max_concurrent_tasks": 4,
    "installed_software": {
      "python": "3.12",
      "cuda": "12.4"
    },
    "capabilities": {
      "hostname": "node-01.internal",
      "s3_config_hash": "sha256:7f83b165..."
    },
    "created_at": 1717070000000,
    "envelope_ts": 1717070000000,
    "nonce": "0194e3b1-7a20-7f91-81b0-95123456789a",
    "metadata": { "environment": "production" }
  }
  ```
- **Responses:**
  - `200 OK`: Registered successfully.
  - `400 Bad Request`: Schema validation failure.
  - `401 Unauthorized`: Token invalid or missing.

---

## 2. Heartbeat & Control Directives

Workers periodically emit heartbeats to report load telemetry and renew active task leases. The Orchestrator's response delivers control directives, such as task cancellations or drain requests.

- **Endpoint:** `PATCH /_worker/workers/{worker_id}`
- **Headers:** `Content-Type: application/json`, `X-Worker-Token: <token>`
- **Request Body (JSON):**
  ```json
  {
    "worker_id": "worker-gpu-node-01",
    "status": "active",
    "usage": {
      "cpu_load_percent": 24.5,
      "ram_used_gb": 12.4,
      "devices_usage": [
        {
          "unit_id": "gpu-0",
          "load_percent": 82.0,
          "vram_used_gb": 18.2,
          "metrics": { "temperature_c": 68 }
        }
      ]
    },
    "current_tasks": ["uuid-task-456"],
    "available_skills": ["transcribe"],
    "hot_skills": ["whisper-large-v3"],
    "available_credits": 3,
    "envelope_ts": 1717070030000,
    "nonce": "0194e3b1-7a3e-7a1b-90f1-abcdef123456"
  }
  ```
- **Success Response (`200 OK`):**
  ```json
  {
    "status": "ok",
    "cancel_requested": ["uuid-task-cancelled-789"],
    "require_full_sync": false,
    "drain_requested": false
  }
  ```

---

## 3. Claiming a Task (Long-Polling & Leases)

Workers claim tasks using an atomic `POST` request. If no matching tasks are available, the Orchestrator holds the connection open up to `timeout` seconds.

- **Primary Endpoint:** `POST /_worker/workers/{worker_id}/tasks/claim?timeout=30`
- **Legacy Deprecated Alias:** `GET /_worker/workers/{worker_id}/tasks/next?timeout=30`
- **Responses:**
  - **`200 OK` (Task Dispatched):**
    ```json
    {
      "job_id": "uuid-job-123",
      "task_id": "uuid-task-456",
      "attempt": 1,
      "job_epoch": 1,
      "type": "transcribe",
      "params": {
        "audio_url": "blob://recordings/meeting.mp3",
        "language": "en"
      },
      "policy": {
        "allowed_skills": ["transcribe", "summarize"],
        "budget_cap": 10.0,
        "budget_allocated": 0.50,
        "budget_currency": "USD"
      },
      "holarchy_depth": 2,
      "transition_count": 5,
      "priority": 1.0,
      "created_at": 1717070000000,
      "envelope_ts": 1717070001000,
      "nonce": "0194e3b1-7b00-741a-85ef-123456789abc",
      "deadline": 1717070300000,
      "lease_ttl_s": 60,
      "orphan_policy": "abort",
      "retry_budget": 3,
      "trace_context": {
        "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
      },
      "security": {
        "signer_id": "apex-orchestrator",
        "orchestrator_signature": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      },
      "metadata": { "priority_class": "production" }
    }
    ```
  - **`204 No Content`:** Long-polling timeout expired with an empty queue. Worker should immediately repoll (subject to jitter).
  - **`429 Too Many Requests` / `503 Service Unavailable`:** Overload or restart; worker **MUST** honor the `Retry-After` header.

---

## 4. Submitting a Result

- **Endpoint:** `POST /_worker/tasks/result`
- **Request Body (JSON):**
  ```json
  {
    "job_id": "uuid-job-123",
    "task_id": "uuid-task-456",
    "attempt": 1,
    "job_epoch": 1,
    "worker_id": "worker-gpu-node-01",
    "status": "success",
    "data": {
      "text": "Hello world from transcribing worker.",
      "duration_s": 42.5
    },
    "costs": {
      "tokens": 1420,
      "compute_seconds": 3.5,
      "estimated_cost": 0.045
    },
    "trace_context": {
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-af34902b70f067aa-01"
    },
    "created_at": 1717070060000,
    "envelope_ts": 1717070060000,
    "nonce": "0194e3b1-7b50-711e-8221-fedcba987654",
    "security": {
      "signer_id": "worker-gpu-node-01",
      "signature": "6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b"
    }
  }
  ```
- **Responses:**
  - `200 OK`: Result accepted and committed.
  - `400 Bad Request`: Result payload failed skill's `output_schema` validation.
  - `409 Conflict`: Obsolete `attempt` or `job_epoch` (fenced out by a newer attempt).
  - `410 Gone`: Job cancelled or deadline expired; worker **MUST NOT** retry.

---

## 5. Worker-Initiated Events (Signals & Bubbling)

- **Endpoint:** `POST /_worker/events`
- **Request Body (JSON):**
  ```json
  {
    "event_id": "0194e3b1-7c00-721a-981a-0123456789ab",
    "worker_id": "worker-gpu-node-01",
    "origin_worker_id": "worker-gpu-node-01",
    "event_type": "progress",
    "payload": {
      "progress_pct": 50.0,
      "stage": "acoustic_model_pass_2"
    },
    "origin_task_id": "uuid-task-456",
    "bubbling_chain": ["worker-gpu-node-01"],
    "provenance_chain": [
      {
        "hop_id": "worker-gpu-node-01",
        "ts": 1717070045000,
        "sig": "3a7b9c..."
      }
    ],
    "bubbling_ttl": 5,
    "target_job_id": "uuid-job-123",
    "trace_context": {
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-123456789abcdef0-01"
    },
    "created_at": 1717070045000,
    "envelope_ts": 1717070045000,
    "nonce": "0194e3b1-7c10-741a-981a-abcdef012345"
  }
  ```
- **Success Response (`200 OK`):**
  ```json
  {
    "status": "accepted",
    "throttle_ms": 0,
    "drop_types": []
  }
  ```

---

## 6. HTTP Status Code Reference Table

| Code | Status | Meaning for Worker |
|---|---|---|
| `200` | OK | Success; proceed with execution. |
| `204` | No Content | Queue empty on claim; wait/poll again immediately. |
| `400` | Bad Request | Schema validation failure; do not retry without fixing payload. |
| `401` | Unauthorized | Token expired or invalid; re-authenticate via STS. |
| `403` | Forbidden | Worker ID mismatch or policy violation. |
| `409` | Conflict | Fencing token mismatch; task superseded by newer attempt. |
| `410` | Gone | Task deadline exceeded or job cancelled; drop task immediately. |
| `429` | Too Many Requests | Rate limit hit; parse `Retry-After` header and wait. |
| `503` | Service Unavailable | Orchestrator unavailable; apply exponential backoff with full jitter. |
