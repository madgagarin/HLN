# Canonical JSON Specification (RFC 8785 / JCS)

**Status:** Normative Standard  
**Editor:** Dmitry Gagarin (madgagarin)  
**Standard Reference:** IETF RFC 8785 (JSON Canonicalization Scheme)  

---

## 1. Requirement & Rationale

In distributed systems employing cryptographic signatures, hash chains, and evidence verification across heterogeneous runtimes (Python, Rust, Go, TypeScript), standard `json.dumps()` or `JSON.stringify()` creates non-deterministic byte representations. Discrepancies in whitespace, key ordering, float representations, and Unicode escape sequences invalidate HMAC signatures and break D-DNA audit chains.

HLN mandates that **ALL** data structures subjected to cryptographic signing, hashing, or MAC calculation **MUST** be canonicalized strictly in compliance with **RFC 8785 (JSON Canonicalization Scheme — JCS)**.

---

## 2. JCS Normalization Rules

A compliant canonicalizer **MUST** enforce the following rules:

### 2.1. Whitespace
- No whitespace **MUST** be emitted outside of string literals.
- Separators **MUST** be literal `,` and `:` without leading or trailing spaces.

### 2.2. Object Key Ordering
- Object member keys **MUST** be sorted lexicographically by their UTF-16 code unit values.
- For code points outside the Basic Multilingual Plane (supplementary characters $\ge \text{U+10000}$), sorting compares surrogate pair code units.

### 2.3. Number Formatting
Numbers **MUST** be formatted according to the ECMAScript 6+ standard (IEEE 754 double precision):
- No leading plus sign (`+`).
- No trailing `.0` for integers (`42`, never `42.0`).
- Scientific notation (`e` or `E`) **MUST** use lowercase `e` with no redundant leading zeros in the exponent.
- Negative zero (`-0.0`) **MUST** be serialized as `0`.
- Values of `NaN`, `Infinity`, and `-Infinity` are **PROHIBITED** in canonical JSON and **MUST** cause a serialization failure.

### 2.4. String Escaping & Unicode Encoding
- The output **MUST** be encoded in UTF-8 without a Byte Order Mark (BOM).
- Only two characters **MUST** be escaped using backslash: quotation mark (`\"`) and reverse solidus (`\\`).
- Control characters (U+0000 through U+001F) **MUST** be escaped using two-character escape sequences (`\b`, `\t`, `\n`, `\f`, `\r`) or six-character Unicode escapes (`\u00XX`).
- All other Unicode characters (including emoji, Cyrillic, CJK) **MUST** be emitted directly as unescaped UTF-8 bytes.

---

## 3. Scope of Application in HLN

RFC 8785 JCS canonicalization **MUST** be applied before hashing or signing the following structures:

1. **`TaskPayload` Signing:** The canonical serialization of the task envelope.
2. **`TaskResult` Signing:** The canonical serialization of the result envelope.
3. **`WorkerEventPayload` Signing:** The canonical serialization of worker events.
4. **`JobState` Anti-Tamper Signing:** The canonical serialization of the signed job state subset.
5. **`D-DNA` Audit Records:** The canonical serialization of each history event before keyed HMAC calculation.
6. **`provenance_chain` Hop Verification:** The canonical serialization of intermediate bubbling records.

---

## 4. Test Vectors

Compliant implementations **MUST** pass the byte-level test vectors published in `conformance/test-vectors/jcs.json`.
