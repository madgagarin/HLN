# HLN Blueprint & FSM Specification

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Conformance:** RFC 2119 / RFC 8174  

---

## 1. Scope & Purpose

This specification defines the syntax, lifecycle, state machine transitions, and execution contracts of the **Blueprint** within the Holarchical Logic Network (HLN). 

A Blueprint is the declarative, compiled description of a deterministic workflow process modeled as a Finite State Machine (FSM). While executing Holons (Ghosts) may incorporate non-deterministic components (such as Large Language Models or human operators), the Blueprint control plane **MUST** remain strictly deterministic, auditable, and bounded.

---

## 2. Terminology & RFC 2119 Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here.

- **State:** A distinct logical stage in the execution lifecycle of a job, represented as a string identifier.
- **Transition:** An atomic, recorded progression from a source state to a target state.
- **Decider:** An advisory or constrained routing node where a bounded set of candidate target states is evaluated under a strict fallback timeout.
- **Step Scope:** The cascading variable-resolution hierarchy accessible to step handlers and conditional routing expressions.
- **Storage Zones:** The five isolated data partitions within every job state (`input`, `output`, `steps`, `history`, `meta`).

---

## 3. Declarative Blueprint Structure

A compliant Blueprint **MUST** be serializable to a canonical JSON representation conforming to the schema below.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://raw.githubusercontent.com/madgagarin/HLN/main/schemas/blueprint.schema.json",
  "title": "HLN Blueprint",
  "type": "object",
  "required": ["name", "start_state", "states", "end_states"],
  "properties": {
    "name": {
      "type": "string",
      "pattern": "^[a-z0-9_-]{1,64}$",
      "description": "Unique canonical identifier of the blueprint."
    },
    "api_version": {
      "type": "string",
      "default": "v1",
      "description": "API version prefix for auto-exposed HTTP endpoints."
    },
    "api_endpoint": {
      "type": "string",
      "description": "Optional REST path mapping for job instantiation (e.g., '/jobs/orders')."
    },
    "start_state": {
      "type": "string",
      "description": "The initial entry state of the FSM."
    },
    "end_states": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "description": "Set of terminal states that complete job execution."
    },
    "input_schema": {
      "type": "object",
      "description": "JSON Schema (Draft 2020-12) validating the initial job input payload."
    },
    "output_schema": {
      "type": "object",
      "description": "JSON Schema (Draft 2020-12) validating the final terminal output payload."
    },
    "events_schema": {
      "type": "object",
      "additionalProperties": { "type": "object" },
      "description": "Per-event-type schemas for worker-initiated signals."
    },
    "states": {
      "type": "object",
      "additionalProperties": {
        "$ref": "#/$defs/StateDefinition"
      }
    },
    "data_stores": {
      "type": "object",
      "additionalProperties": { "type": "object" },
      "description": "Read-only or partitioned state lookup dictionaries."
    }
  },
  "additionalProperties": false,
  "$defs": {
    "StateDefinition": {
      "type": "object",
      "required": ["type"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["start", "step", "decide", "parallel", "sub_blueprint", "end"]
        },
        "on_success": { "type": "string" },
        "on_failure": { "type": "string" },
        "skill": { "type": "string" },
        "timeout_seconds": { "type": "number", "minimum": 0.1 },
        "candidates": {
          "type": "array",
          "items": { "type": "string" }
        },
        "fallback": { "type": "string" },
        "risk": {
          "type": "string",
          "enum": ["low", "medium", "high", "critical"],
          "default": "low"
        },
        "permissions": {
          "type": "array",
          "items": { "type": "string" },
          "default": []
        },
        "conditions": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["target_state", "predicate"],
            "properties": {
              "target_state": { "type": "string" },
              "predicate": { "type": "string" }
            }
          }
        }
      },
      "additionalProperties": false
    }
  }
}
```

---

## 4. The 5 Storage Zones of Job State

Every running job instance governed by a Blueprint **MUST** maintain five strictly partitioned storage zones:

1. **`input` (Immutable Client Payload):**
   Contains the pristine payload provided at job instantiation. It **MUST NOT** be mutated by any subsequent step handler.

2. **`steps` (Execution Blackboard):**
   A dictionary indexed by state name (`steps[state_name]`), recording the outputs returned by executed steps and worker tasks. Steps can read previous step results through `StepScope`.

3. **`output` (Terminal Result):**
   The finalized result computed by the terminal `@bp.end` step. Validated strictly against the Blueprint's `output_schema`.

4. **`history` (Audit Log & D-DNA):**
   Append-only chronological array of transitions, timestamps, step receipts, and cryptographic evidence hashes.

5. **`meta` (Execution Context & Controls):**
   System metadata including `priority`, `policy`, `holarchy_depth`, `parent_job_id`, `trace_context`, and headers.

---

## 5. Step Execution Semantics & Types

### 5.1. Start Step (`@bp.start`)
- Every Blueprint **MUST** declare exactly one `start_state`.
- The start step handler receives the validated `input` payload.
- Upon completion, the handler **MUST** return a target state or `Next(target_state, data=...)`.

### 5.2. Standard Step & Remote Skill Dispatch (`@bp.step`)
- A step handler **MAY** execute local computation or dispatch a task to a remote worker skill via `ctx.dispatch_task(skill_name, params=...)` or `Skill(name, ...)(...)`.
- If an `on_success` target state is defined, successful handler completion automatically triggers a transition to `on_success`.
- If an unhandled exception occurs or a worker reports `status = "failure"`, execution transitions to `on_failure` (if declared) or enters `failed` status.

### 5.3. Advisory Decision Routing (`@bp.decide`)
When dynamic or model-driven routing is required, the Blueprint **MUST** constrain choices via a Decider step:
- The decider declares an explicit whitelist of `candidates: [State_A, State_B, ...]`.
- A mandatory `fallback: State` **MUST** be declared.
- A mandatory `timeout_seconds` (RECOMMENDED: $\le 3.0$ seconds) **MUST** be enforced.
- **Host-Enforced Invariant:** If the decider logic returns a state not in `candidates`, or if the execution times out, the Orchestrator **MUST** force transition to the declared `fallback` state, generating an `ExecutionReceipt` recorded in `history`.

### 5.4. Parallel Execution & DAG Topology (`ctx.dispatch_parallel`)
- When a state initiates concurrent tasks across multiple workers, it enters `waiting_for_parallel_tasks`.
- Each concurrent branch executes under a distinct `task_id` linked to the originating `job_id`.
- The parent job **MUST NOT** transition until all branches conclude or the earliest phase timeout expires.
- Branch results are accumulated into `steps[state_name]` as a dictionary keyed by branch task ID.

### 5.5. Sub-Blueprint Execution (`ctx.run_blueprint`)
- A Blueprint state **MAY** invoke another registered Blueprint as a child Job.
- The child Job inherits:
  - `child.holarchy_depth = parent.holarchy_depth + 1`
  - Attenuated `policy` ($\text{allowed\_skills}_{\text{child}} \subseteq \text{allowed\_skills}_{\text{parent}}$)
  - Clamped `deadline` and divided `budget_allocated`.
- If `child.holarchy_depth > max_holarchy_depth`, the dispatch **MUST** be rejected with quarantine.

---

## 6. StepScope Resolution Matrix

When evaluating conditional transitions or injecting parameters into step handlers, variables **MUST** resolve according to the following strict precedence cascade ($O(1)$ lookups):

```
Priority 1: Current incoming data (most recent step or worker task output)
Priority 2: Job initial input payload (`input.*`)
Priority 3: Context properties (`job_id`, `current_state`, `policy`)
Priority 4: Explicit scope prefixes:
            - `input.<field>`
            - `steps.<state_name>.<field>`
            - `meta.<field>`
            - `data_stores.<store_name>.<key>`
```

---

## 7. Transition Counting & Loop Prevention

To guarantee deterministic termination:
1. Every state transition (including loops and self-transitions) **MUST** atomically increment the job's `transition_count`.
2. The Orchestrator **MUST** evaluate `transition_count` against `MAX_TRANSITIONS_PER_JOB` (default: 50).
3. If `transition_count > MAX_TRANSITIONS_PER_JOB`, the job **MUST** immediately halt and transition to `quarantined` with error code `LIMIT_EXCEEDED_ERROR`.
4. `transition_count` is independent of `holarchy_depth`:
   - `transition_count` prevents horizontal infinite loops ($A \to B \to A \to B$).
   - `holarchy_depth` prevents vertical recursive spawning ($A \to \text{spawn } B \to \text{spawn } A$).
