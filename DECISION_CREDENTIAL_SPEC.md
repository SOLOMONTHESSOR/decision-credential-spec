# Thessor Decision Credential Specification

**Version:** 1.0.0  
**Status:** Draft for Public Review  
**Date:** 2026-06-19  
**Maintainer:** Thessor, Inc.  
**License:** CC BY 4.0

---

## 1. Preamble

Artificial intelligence systems produce outputs — classifications, recommendations, prior authorization decisions, diagnostic flags — but do not produce evidence. At the moment of inference, the model version in effect, the policy applied, the exact output generated, and the operator context are transient. Nothing in a standard inference pipeline creates a tamper-evident record that an auditor, regulator, or adverse-event investigator can verify months or years later.

The Content Provenance and Authenticity (C2PA) standard solved this problem for media: a C2PA manifest cryptographically binds an image or video to metadata describing its origin, provenance chain, and modifications, so downstream consumers can verify that the content is what the publisher claims. The manifest travels with the content.

This specification defines the Thessor Decision Credential: a machine-verifiable, cryptographically signed record of an AI system's decision at the moment of inference. A Decision Credential is to an AI output what a C2PA manifest is to a media asset — a tamper-evident provenance record that can be verified by any party who holds the signer's public key, without access to Thessor's infrastructure.

The format is open. Any AI system, logging pipeline, or compliance toolchain can produce or verify Decision Credentials by implementing this specification.

---

## 2. Decision Credential Object Schema

### 2.1 Envelope (v3)

The core unit of the Decision Credential format. All fields are sealed — covered by the Ed25519 signature — unless marked *(metadata only)*.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorDecisionEnvelope",
  "type": "object",
  "required": [
    "schema_version",
    "signature_suite",
    "decision_id",
    "model_id",
    "model_version",
    "timestamp",
    "input_hash",
    "output",
    "confidence_score",
    "policy_version",
    "canonical_hash",
    "signature"
  ],
  "properties": {
    "schema_version": {
      "type": "string",
      "description": "Envelope schema version. Current value: 'v3'.",
      "enum": ["v1", "v2", "v3"]
    },
    "signature_suite": {
      "type": "string",
      "description": "Cryptographic suite identifier sealed into the envelope. Current value: 'ed25519'.",
      "enum": ["ed25519"]
    },
    "decision_id": {
      "type": "string",
      "description": "Caller-supplied stable identifier for this decision (UUID recommended)."
    },
    "model_id": {
      "type": "string",
      "description": "Identifier for the AI model that produced the decision."
    },
    "model_version": {
      "type": "string",
      "description": "Version string for the model in effect at inference time."
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 UTC timestamp of the inference event."
    },
    "input_hash": {
      "type": "string",
      "description": "SHA-256 hex digest of the canonical input presented to the model."
    },
    "output": {
      "type": ["object", "string", "number", "boolean", "array"],
      "description": "The AI system's output at inference time."
    },
    "confidence_score": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "description": "Model confidence in the output (0.0–1.0)."
    },
    "policy_version": {
      "type": "string",
      "description": "Version identifier for the policy or ruleset applied at inference time."
    },
    "attestations": {
      "type": "object",
      "description": "Optional key-value attestation fields sealed into the envelope.",
      "properties": {
        "regulatory_frameworks": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Keys from the Thessor Regulatory Framework Taxonomy."
        },
        "clinician_npi": { "type": "string" },
        "device_udi": { "type": "string" },
        "patient_id_hash": { "type": "string" },
        "model_registry_id": { "type": "string" },
        "model_artifact_hash": { "type": "string" },
        "base_model_id": { "type": "string" },
        "model_card_url": {
          "type": "string",
          "description": "Excluded from canonical computation (mutable field, not sealed)."
        }
      }
    },
    "canonical_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$",
      "description": "SHA-256 hex digest of the canonical byte string (see Section 3)."
    },
    "signature": {
      "type": "string",
      "description": "Ed25519 signature over canonical_hash bytes, hex-encoded."
    },
    "envelope_id": {
      "type": "string",
      "description": "*(metadata only)* Server-assigned UUID for this envelope record."
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "*(metadata only)* Server-assigned timestamp of envelope creation."
    },
    "key_reference": {
      "type": "string",
      "description": "*(metadata only)* Identifier for the key that produced the signature ('local' or a KMS key ARN)."
    }
  }
}
```

### 2.2 Replay Record

A deterministic replay record binds the exact model artifact and canonical inputs/outputs to the original decision envelope. Sealing a replay record provides machine-verifiable evidence that inference can be reproduced exactly.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorReplayRecord",
  "type": "object",
  "required": [
    "replay_schema_version",
    "decision_id",
    "canonical_inputs_hash",
    "canonical_output_hash",
    "model_artifact_hash",
    "model_version",
    "replay_signature"
  ],
  "properties": {
    "replay_schema_version": {
      "type": "string",
      "description": "Replay record schema version. Current value: 'replay-v1'."
    },
    "decision_id": {
      "type": "string",
      "description": "Envelope ID of the parent decision this replay is registered to."
    },
    "canonical_inputs_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$",
      "description": "SHA-256 hex digest of the canonical input set."
    },
    "canonical_output_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$",
      "description": "SHA-256 hex digest of the canonical output."
    },
    "model_artifact_hash": {
      "type": "string",
      "description": "SHA-256 hex digest of the model artifact (weights + config) in effect."
    },
    "model_version": { "type": "string" },
    "replay_signature": {
      "type": "string",
      "description": "Ed25519 signature over the replay canonical bytes, hex-encoded."
    },
    "replay_id": { "type": "string" },
    "registered_at": { "type": "string", "format": "date-time" }
  }
}
```

### 2.3 Rationale Record

A sealed decision-rationale record captures the model's internal reasoning at inference time: confidence distributions, salient inputs, policy logic applied, and optional human oversight action.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorRationaleRecord",
  "type": "object",
  "required": [
    "rationale_schema_version",
    "decision_id",
    "confidence_distribution",
    "salient_inputs",
    "policy_logic_applied",
    "rationale_hash",
    "rationale_signature"
  ],
  "properties": {
    "rationale_schema_version": {
      "type": "string",
      "description": "Rationale record schema version. Current value: 'rationale-v1'."
    },
    "decision_id": { "type": "string" },
    "confidence_distribution": {
      "type": "object",
      "description": "Per-class confidence scores at inference time."
    },
    "salient_inputs": {
      "type": "object",
      "description": "Feature attributions or salient input identifiers."
    },
    "policy_logic_applied": {
      "type": "object",
      "description": "Structured representation of the policy rules applied."
    },
    "human_oversight_action": {
      "type": ["string", "null"],
      "description": "Human operator action taken at or after inference, if any."
    },
    "human_operator_id_hash": {
      "type": ["string", "null"],
      "description": "SHA-256 hash of the operator identifier. Raw identifier is never stored."
    },
    "counterfactual_hint": {
      "type": ["object", "null"],
      "description": "Optional structured counterfactual: what inputs would change the output."
    },
    "rationale_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$"
    },
    "rationale_signature": {
      "type": "string",
      "description": "Ed25519 signature over rationale canonical bytes, hex-encoded."
    },
    "rationale_id": { "type": "string" },
    "sealed_at": { "type": "string", "format": "date-time" }
  }
}
```

### 2.4 Population Attestation

A sealed aggregate performance attestation covering a defined population of decisions over a defined time window. Required for surveillance-grade compliance (GMLP, EU AI Act, PCCP, ISO 42001).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorPopulationAttestation",
  "type": "object",
  "required": [
    "attestation_schema_version",
    "model_version",
    "period_start",
    "period_end",
    "attestation_hash",
    "attestation_signature"
  ],
  "properties": {
    "attestation_schema_version": {
      "type": "string",
      "description": "Population attestation schema version. Current value: 'population-v1'."
    },
    "model_version": { "type": "string" },
    "site_identifier_hash": { "type": ["string", "null"] },
    "period_start": { "type": "string", "format": "date-time" },
    "period_end": { "type": "string", "format": "date-time" },
    "decision_count": { "type": ["integer", "null"], "minimum": 0 },
    "confirmed_count": { "type": ["integer", "null"], "minimum": 0 },
    "true_positive_count": { "type": ["integer", "null"], "minimum": 0 },
    "false_positive_count": { "type": ["integer", "null"], "minimum": 0 },
    "true_negative_count": { "type": ["integer", "null"], "minimum": 0 },
    "false_negative_count": { "type": ["integer", "null"], "minimum": 0 },
    "sensitivity": { "type": ["number", "null"], "minimum": 0.0, "maximum": 1.0 },
    "specificity": { "type": ["number", "null"], "minimum": 0.0, "maximum": 1.0 },
    "drift_flag": { "type": "boolean" },
    "regulatory_frameworks": {
      "type": "array",
      "items": { "type": "string" }
    },
    "attestation_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$"
    },
    "attestation_signature": {
      "type": "string",
      "description": "Ed25519 signature over population attestation canonical bytes."
    },
    "attestation_id": { "type": "string" },
    "attested_at": { "type": "string", "format": "date-time" }
  }
}
```

### 2.5 Merkle Checkpoint

A Merkle checkpoint anchors a contiguous range of decision envelope IDs into a single root hash, enabling set-membership proof without disclosing envelope contents.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorMerkleCheckpoint",
  "type": "object",
  "required": ["checkpoint_id", "root_hash", "seal_count", "computed_at"],
  "properties": {
    "checkpoint_id": { "type": "string" },
    "root_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$",
      "description": "Merkle root over ordered SHA-256 digests of sealed envelope IDs."
    },
    "seal_count": {
      "type": "integer",
      "minimum": 1,
      "description": "Number of envelopes included in this checkpoint."
    },
    "computed_at": { "type": "string", "format": "date-time" }
  }
}
```

### 2.6 Audit Export Packet

A self-contained, offline-verifiable bundle containing all records associated with a single decision. The `export_signature` seals the entire assembled packet.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12",
  "title": "ThessorAuditExportPacket",
  "type": "object",
  "required": [
    "export_schema_version",
    "exported_at",
    "decision",
    "public_key_hex",
    "key_reference",
    "verification_instructions",
    "export_signature"
  ],
  "properties": {
    "export_schema_version": { "type": "string", "enum": ["v1"] },
    "exported_at": { "type": "string", "format": "date-time" },
    "decision": { "$ref": "#/$defs/ThessorDecisionEnvelope" },
    "replay": {
      "oneOf": [{ "$ref": "#/$defs/ThessorReplayRecord" }, { "type": "null" }]
    },
    "rationale": {
      "oneOf": [{ "$ref": "#/$defs/ThessorRationaleRecord" }, { "type": "null" }]
    },
    "chain": {
      "type": ["object", "null"],
      "description": "Consequence chain: the decision envelope plus linked outcome envelopes."
    },
    "population_context": {
      "oneOf": [{ "$ref": "#/$defs/ThessorPopulationAttestation" }, { "type": "null" }]
    },
    "merkle_checkpoint": {
      "oneOf": [{ "$ref": "#/$defs/ThessorMerkleCheckpoint" }, { "type": "null" }]
    },
    "public_key_hex": {
      "type": "string",
      "description": "Hex-encoded Ed25519 public key that produced all signatures in this packet."
    },
    "key_reference": { "type": "string" },
    "verification_instructions": {
      "type": "object",
      "properties": {
        "algorithm": { "type": "string" },
        "signature_suite": { "type": "string" },
        "steps": { "type": "array", "items": { "type": "string" } }
      }
    },
    "export_signature": {
      "type": "string",
      "description": "Ed25519 signature over SHA-256(canonical(packet minus export_signature)), hex-encoded."
    }
  }
}
```

---

## 3. Canonicalization Rules

The canonical byte string is the input to the SHA-256 hash that becomes `canonical_hash`. The same algorithm is used for all record types in this specification. Implementations MUST produce byte-identical output to pass signature verification.

**Algorithm (language-agnostic):**

1. **Sort keys recursively.** For every JSON object at any depth, sort the keys lexicographically (Unicode code point order). Apply this recursively to all nested objects.
2. **Normalize strings.** Strip leading and trailing ASCII whitespace (space, tab, newline, carriage return) from all string values. Do not strip whitespace from string keys.
3. **Preserve other types.** Numbers, booleans, arrays, and null values are not modified. Array element order is preserved.
4. **Serialize to compact JSON.** Serialize the normalized structure to UTF-8 bytes using compact JSON (no spaces after `:` or `,`). The character encoding MUST be UTF-8. Non-ASCII characters MUST NOT be escaped as `\uXXXX` sequences unless they are control characters below U+0020.

**v3 envelope canonicalization specifics:**

For a v3 envelope, the canonical input is constructed as follows before applying the algorithm above:

- **Without attestations:** wrap the decision fields in `{"signature_suite": "<suite>", ...decision_fields}` where the `signature_suite` field sorts before other fields whose keys begin with a letter after `s`.
- **With attestations (minus `model_card_url`):** wrap as `{"payload": {...decision_fields}, "attestations": {...attestation_fields}, "signature_suite": "<suite>"}`.

The exact structure is determined at seal time and must be reproduced identically at verification time using the stored `seal_version`, `signature_suite`, and attestation data.

---

## 4. Signature Verification Algorithm

**Prerequisites:**
- The signer's Ed25519 public key (32 bytes, hex-encoded as a 64-character lowercase hex string).
- The Decision Credential object or export packet to verify.

**Steps:**

1. Extract `canonical_hash` from the envelope.
2. Extract `signature` (hex-encoded Ed25519 signature, 64 bytes / 128 hex chars).
3. Extract `public_key_hex` from the envelope or from an out-of-band trusted source.
4. Reconstruct the canonical input from the original decision fields using the rules in Section 3. The `seal_version` field determines which canonicalization variant to apply.
5. Compute `SHA-256(canonical_bytes)` → `computed_hash` (32 bytes, hex-encoded).
6. Assert `computed_hash == canonical_hash`. If not equal, the envelope has been tampered with or the canonical input was not correctly reconstructed.
7. Decode `public_key_hex` to 32 bytes.
8. Decode `signature` to 64 bytes.
9. Verify the Ed25519 signature: `Ed25519Verify(public_key, message=canonical_hash.encode("utf-8"), signature)`. The message passed to the signature algorithm is the SHA-256 hex digest string encoded as UTF-8 bytes (not the raw 32-byte hash).
10. If step 6 and step 9 both pass, the envelope is valid.

**For export packet verification:**

Apply the above to `decision.canonical_hash` and `decision.signature`. Then verify the `export_signature` by computing `SHA-256(canonical(packet_without_export_signature))` and verifying the Ed25519 signature over those 32 bytes encoded as a hex string in UTF-8.

---

## 5. Extension Points and Versioning

### 5.1 Schema Version Pattern

Each record type carries a `schema_version` (or `{type}_schema_version`) field that is included in the canonical bytes and therefore covered by the signature. This ensures that a verifier can unambiguously determine which canonicalization and field set applies to any given record.

**Versioning rules:**

- **Additive changes** (new optional fields): increment the minor version suffix (e.g., `v3` → `v3.1`). Older verifiers that do not recognize new fields will still produce the correct canonical hash if the new fields are excluded from the canonical computation by convention. New fields added in a minor version MUST be appended in canonical sort order.
- **Breaking changes** (removed fields, changed semantics, new required fields): increment the major version (e.g., `v3` → `v4`). Implementations MUST reject envelopes whose `schema_version` they do not recognize.

### 5.2 Signature Suite Extension

The `signature_suite` field is sealed into every v3 envelope. Introducing a new cryptographic algorithm requires:

1. Defining a new suite identifier string (e.g., `ed25519-dilithium2`).
2. Publishing a suite specification (this document or an addendum).
3. Incrementing the envelope `schema_version`.

Existing v3 envelopes remain verifiable under `ed25519` indefinitely.

### 5.3 Attestation Key Extension

The Regulatory Framework Taxonomy (Section 8) defines the set of valid `regulatory_frameworks` key strings. New frameworks are added to the taxonomy by publishing a new taxonomy version. Attestation keys not in the taxonomy are rejected by compliant implementations.

---

## 6. Conformance Levels

Implementations producing or consuming Decision Credentials MUST declare one of the following conformance levels.

### Level 1: Decision Seal

**Required records:** Decision Envelope (v3)

**Minimum fields sealed:**
- `decision_id`, `model_id`, `model_version`, `timestamp`, `input_hash`, `output`, `confidence_score`, `policy_version`
- `signature_suite` (v3 envelopes)
- `canonical_hash`, `signature`

**Verification capability:** A Level 1 verifier can confirm that a decision record was produced by a specific model version at a specific time and has not been altered since sealing.

**Use cases:** Baseline audit trail, 21 CFR Part 11 electronic record requirements, CMS-4201-F prior authorization logging.

---

### Level 2: Decision Seal + Replay + Rationale

**Required records:** Level 1 + Replay Record + Rationale Record

**Additional fields sealed:**
- Replay: `canonical_inputs_hash`, `canonical_output_hash`, `model_artifact_hash`, `replay_signature`
- Rationale: `confidence_distribution`, `salient_inputs`, `policy_logic_applied`, `rationale_signature`

**Verification capability:** Level 2 adds deterministic reproducibility evidence and machine-readable explainability. An auditor can verify that the model artifact in effect at inference time matches the declared version, and that the stated reasoning is bound to the decision.

**Use cases:** FDA PCCP model-change audit trails, ONC HTI-1 DSI source attribute documentation, NIST AI RMF MEASURE function.

---

### Level 3: Decision Seal + Replay + Rationale + Population Attestation + Merkle Checkpoint + Audit Export Packet

**Required records:** Level 2 + Population Attestation + Merkle Checkpoint + Audit Export Packet

**Additional fields sealed:**
- Population Attestation: `model_version`, `period_start`, `period_end`, confusion matrix counts, `sensitivity`, `specificity`, `drift_flag`, `attestation_signature`
- Merkle Checkpoint: `root_hash`, `seal_count`
- Export Packet: `export_signature` over the assembled bundle

**Verification capability:** Level 3 adds surveillance-grade population performance evidence, Merkle-anchored set membership proof (a decision cannot be silently removed from the ledger), and a fully self-verifying offline export packet that a regulator can verify without access to Thessor's infrastructure.

**Use cases:** EU AI Act high-risk AI post-market surveillance, ISO/IEC 42001 continual improvement evidence, GMLP Principles 7 and 9, FDA PCCP total-product-lifecycle monitoring.

---

## 7. Reference Implementation

The reference implementation is the Thessor API and SDK:

- **API source:** `api.py`, `cnds_core.py`, `ledger.py`, `regulatory_taxonomy.py` in this repository.
- **Python SDK:** `sdk/thessor/` — zero-dependency client, MCP middleware, FHIR/DICOM/HL7/ATNA connectors.
- **MCP middleware:** `sdk/thessor/mcp.py` — `ThessorMCPMiddleware` for automatic sealing of agent tool calls.

The reference implementation produces Level 3 conformant credentials when all optional record types are used. Level 1 and Level 2 subsets are supported by using only the relevant endpoints.

---

## 8. Regulatory Crosswalk Table

The following table maps conformance levels to the regulatory requirements they satisfy. "Partial" indicates that the credential provides evidence for the cited requirement but may not be the sole control required for full compliance.

| Regulation / Framework | Level 1 | Level 2 | Level 3 | Notes |
|---|---|---|---|---|
| **21 CFR Part 11** — Electronic Records & Signatures | Full | Full | Full | Tamper-evident sealed envelope satisfies the audit trail requirement. |
| **FDA PCCP** — Predetermined Change Control Plan | Partial | Full | Full | Level 2 adds model artifact hash, enabling SaMD version-change traceability per Aug 2025 Final Guidance. |
| **ONC HTI-1** — Predictive DSI Source Attributes | Partial | Full | Full | Level 2 rationale record covers the explainability and human oversight source attributes. |
| **CMS-4201-F** — Prior Authorization Auditability | Full | Full | Full | Level 1 provides the sealed utilization management audit record. |
| **EU AI Act** — High-Risk AI (Annex III) | Partial | Partial | Full | Level 3 population attestation satisfies post-market monitoring (Article 72). Merkle checkpoint provides immutability evidence. |
| **GMLP Principle 7** — Human-AI Team Performance | Partial | Full | Full | Level 2 human_oversight_action field documents operator involvement. Level 3 adds population performance evidence. |
| **GMLP Principle 9** — Clear Essential Information | Full | Full | Full | Tamper-evident output provenance satisfies the clear information requirement. |
| **NIST AI RMF** — GOVERN / MAP / MEASURE / MANAGE | Partial | Full | Full | Level 2 covers MEASURE (confidence, rationale); Level 3 adds GOVERN evidence via population attestation. |
| **ISO/IEC 42001:2023** — AI Management System | Partial | Partial | Full | Level 3 satisfies Clause 6.1 (risk evidence), Clause 9.1 (monitoring and measurement), and Clause 10.1 (continual improvement via population attestation time series). |
| **HTI-1** — Evidence-Based DSI Source Attributes | Partial | Full | Full | Level 2 policy_logic_applied field captures the evidence-base reference. |

---

*This document is released under the Creative Commons Attribution 4.0 International License. Implementers are encouraged to submit errata and proposed extensions via the Thessor open-standards repository.*
