# RXON Protocol Specification

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Conformance:** RFC 2119 / RFC 8174  

---

## 1. Scope & Protocol Versioning

The Reverse Axon (RXON) protocol defines the message models, validation schemes, cryptographic bindings, and transport interactions between orchestrators (`holarchy`), execution workers (`holonode`), registries (`holoreg`), and developer tooling (`hln`).

### 1.1. Protocol Version Negotiation (Handshake)

All communicating peers negotiate the active protocol version using explicit version range advertisements:

1. **Protocol Constants:**
   * Current Specification Version: `1.1.0`
   * Minimum Supported Version: `1.0.0`
   * HTTP Header: `X-RXON-Version`

2. **Handshake via Worker Registration:**
   When a worker node connects to an orchestrator, its `WorkerRegistration` payload advertises:
   * `protocol_version`: Current preferred version (e.g. `"1.1.0"`).
   * `protocol_min_version`: Minimum acceptable protocol version (e.g. `"1.0.0"`).
   * `protocol_max_version`: Maximum acceptable protocol version (e.g. `"1.1.0"`).

3. **Orchestrator Negotiation Rules:**
   Let $[\text{client.min}, \text{client.max}]$ be the client's supported version range, and $[\text{server.min}, \text{server.max}]$ be the orchestrator's range.
   $$\text{CommonVersions} = [\text{client.min}, \text{client.max}] \cap [\text{server.min}, \text{server.max}]$$
   * If $\text{CommonVersions} \ne \emptyset$, the orchestrator MUST select $\max(\text{CommonVersions})$ as the active session version, return it in the registration response, and include `X-RXON-Version: <version>` on subsequent responses.
   * If $\text{CommonVersions} = \emptyset$, the orchestrator MUST reject the registration with HTTP status code `426 Upgrade Required` and JSON error details:
     ```json
     {
       "error": {
         "code": "PROTOCOL_MISMATCH_ERROR",
         "message": "Protocol version mismatch: client supports [1.0.0, 1.0.0], server requires [1.1.0, 1.2.0]",
         "details": {
           "server_min": "1.1.0",
           "server_max": "1.2.0"
         }
       }
     }
     ```
   * Legacy v1.0.0 clients omitting `protocol_min_version` and `protocol_max_version` MUST be treated as advertising $[\text{"1.0.0"}, \text{"1.0.0"}]$.

---

## 2. Canonical Skill Identifiers & Namespaces

To prevent naming collisions in multi-tenant and federated topologies, skill names support hierarchical namespacing.

### 2.1. Skill Name Syntax

A valid skill name MUST conform to the regular expression:
```regex
^[a-zA-Z0-9_-]+(/[a-zA-Z0-9_-]+)?$
```
* Max length: 192 characters.
* Examples of valid names: `ocr`, `ocr-fast`, `vision/ocr-fast`, `corp_kyc/document-verify`.
* Disallowed: leading/trailing slashes (`/vision/ocr`), multiple slashes (`vision/ocr/fast`), special punctuation (`vision@ocr`, `vision:ocr`).

### 2.2. Generic Identifiers

All runtime entity IDs (`worker_id`, `job_id`, `task_id`, `blueprint_name`) MUST conform to:
```regex
^[a-zA-Z0-9_-]+$
```
with maximum length of 128 characters.

---

## 3. Skill Metadata & SemVer Matching (`SkillInfo`)

### 3.1. Data Model

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "name": "vision/ocr-fast",
  "type": "service",
  "description": "High-throughput document OCR",
  "version": "1.4.2+sha256.bbb222098fc1",
  "input_schema": { ... },
  "output_schema": { ... },
  "risk": "low",
  "permissions": ["net:http:outbound"],
  "schema_dialect": "json-schema"
}
```

### 3.2. Version Matching Rules

When matching a task request requirement against registered skills:
1. `name`: MUST match strictly (case-sensitive, including namespace).
2. `type`: If specified in request, MUST match registered skill type.
3. `version`:
   * If `req.version` is omitted or `null`: Match is SUCCESSFUL (any available version).
   * If `req.version == registered.version`: Match is SUCCESSFUL.
   * If strings differ, parse SemVer 2.0.0:
     * Base triplets $(major, minor, patch, prerelease)$ MUST match SemVer ordering precedence.
     * **Build Metadata Stripping:** If `req.version` does NOT contain build metadata (e.g. `1.4.2`), registered build metadata (`+sha256...`) MUST be ignored during matching (i.e. request `1.4.2` matches `1.4.2+sha256.bbb222098fc1`).
     * **Exact Build Metadata Pinning:** If `req.version` explicitly contains build metadata (e.g. pinned from `hln.lock`), strict equality of the build metadata string is required.

---

## 4. Structural Schema Compatibility (`is_schema_compatible`)

Automated contract validation ensures that publishing new versions does not break existing consumers:

### 4.1. Input Schema Contravariance (`mode = "input"`)
The new version MUST accept all inputs accepted by the previous version:
1. All fields marked `required` in the previous schema MUST remain `required` in the new schema.
2. New `required` fields MUST NOT be introduced without a MAJOR SemVer increment.
3. Adding optional fields is permitted as a MINOR or PATCH increment.
4. Restricting type definitions (e.g. `number` $\rightarrow$ `integer`, narrowing `enum` choices, increasing `minimum`) is a breaking change.

### 4.2. Output Schema Covariance (`mode = "output"`)
The new version MUST guarantee all outputs promised by the previous version:
1. All guaranteed fields (`required` and present properties) from the previous schema MUST remain present and guaranteed in the new schema.
2. Removing output fields or making previously guaranteed fields optional is a breaking change (MAJOR).
3. Adding new output fields is permitted as a MINOR increment.

---

## 5. Large Payload Offloading: `BlobReference`

For tasks exchanging binary assets, model weights, or large datasets exceeding transport thresholds (64 KiB), RXON provides the `BlobReference` model:

```json
{
  "uri": "s3://holarchy-payloads/job-123/weights.safetensors",
  "digest": "sha256:7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069",
  "size_bytes": 1145000000,
  "media_type": "application/octet-stream",
  "etag": "\"7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069\""
}
```

* Payloads smaller than 64 KiB SHOULD remain inline within `TaskPayload.params`.
* Payloads $\ge$ 64 KiB MUST be offloaded to Content-Addressable Storage (CAS) or object storage and referenced via `BlobReference`.
* Verification of `digest` using SHA-256 before ingestion by sub-workers is MANDATORY.
