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

This specification defines the **Thessor Decision Credential**: a machine-verifiable, cryptographically signed record of an AI system's decision at the moment of inference. A Decision Credential is to an AI output what a C2PA manifest is to a media asset — a tamper-evident provenance record that can be verified by any party who holds the signer's public key, without access to Thessor's infrastructure.

The format is open. This document defines what a Decision Credential is, the guarantees it provides, and the conformance levels that describe increasing depth of accountability. Full implementer documentation is maintained by Thessor and provided to adopters and design partners.

---

## 2. Governance

This specification is authored and maintained by Thessor, Inc. Thessor stewards the format, its versioning, and the definition of conformance levels and extensions. Proposed changes, new conformance levels, and new sealed record types are reviewed and ratified by Thessor to preserve interoperability across implementations.

The specification is published under CC BY 4.0: any party may read, reference, and implement it. Thessor remains the authoritative source for the canonical version and for the implementer documentation required to produce conformant credentials.

---

## 3. The Problem

An AI decision, once made, leaves no inherent evidence of itself. Consider the questions an auditor or regulator asks after the fact:

- Which exact model version produced this output?
- What policy or clinical criteria were in force at that moment?
- Who, if anyone, reviewed or overrode it?
- What was the eventual outcome, and was the decision bound to it at the time — or reconstructed later?

Standard logging answers none of these in a tamper-evident way. A log entry can be edited. A database row can be altered by anyone with sufficient access. A reconstruction performed during litigation is, by definition, after the fact and contestable.

A Decision Credential answers these questions with a cryptographic guarantee: the record was sealed at the moment of decision, and any alteration to any sealed field is detectable by anyone, using only the signer's public key.

---

## 4. What a Decision Credential Is

A Decision Credential is a structured, cryptographically signed record. At its core it binds together the identity of the decision, the context in which it was made, and a signature that makes the whole record tamper-evident.

A credential may carry, depending on conformance level, the following components. Each is described here at the conceptual level; the precise field definitions are part of the implementer documentation.

**Decision envelope.** The core record. Identifies the decision, the model that produced it, the model version, the policy version, and the output. Carries the signature suite used and the signer's public key, so the record can be verified without trusting the issuer.

**Replay record.** Binds the exact inputs and the exact output of the inference, so that a third party can re-run the model against the sealed inputs and confirm the output matches. This distinguishes proving *which* model ran from proving *what it did*.

**Rationale record.** Seals the explanation of the decision — the confidence distribution, the salient inputs, the policy logic applied, and any human oversight action. Makes the account a system gives of *why* it decided as tamper-evident as the decision itself.

**Consequence chain.** Binds a decision forward to its eventual outcome through sealed causal links. When an outcome is later confirmed — a diagnosis, a reversal on appeal, a clinical result — it is sealed and linked back to the original decision, so the causal relationship is committed at the time, not inferred afterward.

**Population attestation.** A signed, tamper-evident statement of model performance over a population and time window — confirmed-outcome counts and the resulting performance measures. Provides aggregate evidence suitable for post-market surveillance.

**Audit export packet.** A single, self-contained bundle assembling the credential and its associated records, the signer's public key, and verification instructions, sealed with a bundle-level signature. Designed to be filed and verified entirely offline, without access to the issuer's systems.

---

## 5. Guarantees

A conformant Decision Credential provides:

**Tamper-evidence.** Any alteration to any sealed field invalidates the signature. There is no silent edit.

**Independent verifiability.** Verification requires only the signer's public key. No access to the issuer's infrastructure, and no trust in the issuer, is required.

**Decision-time sealing.** The record is created at the moment of inference, not reconstructed later. The timestamp and context are part of what is sealed.

**Cryptographic durability.** The signature suite is itself a sealed, versioned field, so credentials can migrate to stronger signature algorithms — including post-quantum schemes — without invalidating records made under earlier suites.

---

## 6. Architectural Principles

The specification rests on a small set of principles. The mechanics that implement them are part of the implementer documentation.

**Deterministic canonicalization.** Before signing, a record is reduced to a single canonical byte representation, so that the same logical content always produces the same bytes and therefore the same verifiable signature.

**Detached, verifiable signatures.** Each record carries a signature over its canonical bytes. Verification is performed against a published public key using a standard signature algorithm.

**Append-only history.** Records are never mutated. Corrections and confirmations are added as new sealed records that reference the originals.

**Public verification surface.** A credential can be verified by anyone holding the public key, including through a public verification endpoint, without authentication.

**Versioned evolution.** Seal versions, schema versions, and the signature suite are all explicit fields, so the format can evolve while remaining backward-verifiable.

---

## 7. Conformance Levels

Conformance is layered. Each level builds on the one below and signals increasing depth of accountability.

**Level 1 — Decision Seal.**
A signed decision envelope. Proves which model, which version, and which output, sealed at decision time and independently verifiable. This is the minimum conformant credential.

**Level 2 — Decision Seal with Reconstruction and Rationale.**
Level 1 plus a replay record and a rationale record. Proves not only which model decided, but that the decision can be reproduced from its sealed inputs, and that the explanation given for the decision is itself tamper-evident.

**Level 3 — Full Accountability Record.**
Level 2 plus consequence binding, population attestation, external anchoring of the record history, and the offline audit export packet. Provides decision-to-outcome provenance, population-level performance evidence, and a self-contained, regulator-grade artifact.

---

## 8. Regulatory Crosswalk

The Decision Credential is designed to map onto the evidentiary expectations of major AI and health regulatory frameworks. The table indicates the conformance level at which each framework's core record-keeping and accountability expectations are substantively addressed.

| Framework | Addressed at |
|---|---|
| ONC HTI-1 (Predictive DSI transparency) | Level 1 |
| FDA Good Machine Learning Practice (GMLP) | Level 2 |
| 21 CFR Part 11 (electronic records, signatures) | Level 1 |
| FDA Predetermined Change Control Plan (PCCP) | Level 2–3 |
| CMS-4201-F (prior authorization) | Level 1–2 |
| EU AI Act (high-risk record-keeping, human oversight) | Level 2 |
| NIST AI Risk Management Framework | Level 2 |
| ISO/IEC 42001 (AI management system) | Level 3 |

This crosswalk indicates where the credential format provides supporting evidence. It is not a determination of regulatory compliance, which depends on the full obligations of each framework and the practices of the implementing organization.

---

## 9. Reference Implementation

Thessor operates the reference implementation of this specification. Decision Credentials produced by Thessor are verifiable at the public verification endpoint published at thessor.com, and through the Thessor SDK.

Organizations seeking to produce or verify conformant Decision Credentials, or to obtain the full implementer documentation, may contact Thessor.

---

## 10. Contact

Thessor, Inc.
solomon@thessor.com
thessor.com

---

*This document defines the Thessor Decision Credential format at the level required to understand, reference, and evaluate it. Full field-level schemas, canonicalization rules, and the verification algorithm are maintained as implementer documentation by Thessor and provided to adopters and design partners.*
