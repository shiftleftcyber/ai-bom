# Building Trustworthy AI Supply Chain Metadata with AI-SBOMs

Full technical whitepaper  
ShiftLeftCyber AI-BOM proof of concept  
May 28, 2026

## Executive Summary

AI systems create supply chain questions that conventional software inventories do not fully answer. Security and platform teams need to know not only which software packages are present, but also which models are deployed, where they came from, which datasets influenced them, which infrastructure supports inference, which controls are in place, and whether the published metadata can be trusted.

The ShiftLeftCyber AI-BOM proof of concept addresses this problem with a strict AI-SBOM JSON Schema and a SecureSBOM signing and verification workflow. The schema is based on the G7 SBOM for AI minimum-element clusters and is published as version `1.0.0`. It captures metadata, system-level properties, models, datasets, infrastructure, security properties, and key performance indicators. SecureSBOM extends that schema into an operational trust workflow: validate the AI-SBOM, embed a JSON Signature Format author signature, canonicalize the document, sign the canonical payload, distribute the signed artifact, and verify integrity before relying on it.

This paper provides the complete technical argument, implementation framework, risk analysis, evaluation criteria, and recommended approach for teams adopting AI-SBOMs in build, release, procurement, governance, or audit workflows.

## Audience

This paper is written for:

- Security leaders evaluating AI supply chain assurance programs.
- Engineering leaders responsible for AI platform reliability and release governance.
- Security architects designing signing, verification, and trust-policy workflows.
- Platform teams integrating SBOM validation into CI/CD and artifact pipelines.
- Governance, risk, and compliance teams defining evidence requirements for AI systems.
- Product and procurement teams that need repeatable intake criteria for third-party AI components.

The paper assumes familiarity with SBOM concepts, JSON-based APIs, public-key signatures, CI/CD workflows, and basic AI system architecture.

## Background and Context

Traditional SBOMs are designed to describe software components and dependencies. They are valuable for vulnerability management, license review, component inventory, supplier transparency, and incident response. AI systems still need those capabilities, but they also introduce additional supply chain surfaces.

An AI-enabled application may include:

- Conventional application code and open source packages.
- A hosted or self-managed model artifact.
- Training, fine-tuning, evaluation, retrieval, or monitoring datasets.
- Data transformation pipelines and feature stores.
- Prompt templates, policy layers, and guardrail services.
- Inference infrastructure, model gateways, endpoints, accelerators, and runtime dependencies.
- Evaluation metrics related to performance, fairness, safety, reliability, and operational quality.
- Security controls, threat model notes, vulnerability references, and assurance evidence.

These facts often live in separate systems. The model registry may track versions. The data platform may track datasets. The CI pipeline may track application builds. The security team may track controls. The governance team may track approvals. Without a shared machine-readable format, organizations struggle to enforce consistent AI inventory and trust requirements.

The AI-SBOM schema in this proof of concept treats AI supply chain metadata as a strict contract. A valid document can be checked by tools, signed by an authorized producer, distributed with the artifact or release record, and verified before use.

## Problem Statement

Organizations need a way to answer practical AI supply chain questions with repeatable automation:

- What model is this system using, and which version or identifier is authoritative?
- Which datasets are materially connected to the model or system behavior?
- Which infrastructure and runtime services are part of the deployed system?
- What security properties and controls are declared?
- Which operational metrics are available?
- Has the inventory metadata changed since it was signed?
- Was the metadata signed by a key that the organization trusts for this system or workflow?

Existing documentation practices do not reliably answer these questions at scale. Free-form documents are difficult to validate. Spreadsheets drift. Model cards are often useful for human review but inconsistent for release automation. Conventional SBOMs do not consistently model AI-specific metadata.

The required capability is a layered trust workflow:

| Layer | Purpose |
| --- | --- |
| Schema validation | Confirm that the AI-SBOM has the expected structure, required fields, field types, and allowed values. |
| Canonicalization | Convert equivalent JSON representations into stable bytes before signing or verification. |
| Cryptographic signature | Make covered metadata tamper-evident. |
| Trust policy | Decide whether the signing key, certificate, or key identifier is acceptable. |
| Semantic policy | Evaluate whether the declared content satisfies organizational requirements. |

Each layer answers a different question. Schema validity does not prove identity. A valid signature does not prove that the AI system is approved. Policy checks require trustworthy structured metadata, but they should not be collapsed into the schema itself.

## Key Technical Concepts

### AI-SBOM

An AI-SBOM is a machine-readable bill of materials for an AI system. It extends the SBOM idea beyond package dependencies to include model, dataset, infrastructure, security, and KPI metadata.

### Minimum-Element Clusters

The proof-of-concept schema maps to seven clusters from the G7 SBOM for AI minimum-elements work:

| Cluster | Purpose |
| --- | --- |
| Metadata | Identifies the BOM format, producer, version, timestamp, tool context, dependency relationships, and optional author signature. |
| System | Describes the AI system name, components, producer, version, data flow, input and output properties, and intended application area. |
| Models | Records model names, identifiers, versions, producers, hashes, architecture, training properties, licenses, and references. |
| Datasets | Describes dataset identity, provenance, content, hashes, sensitivity, licenses, and dataset relationships. |
| Infrastructure | Documents runtime software, hardware, deployment environment, endpoints, and operational dependencies. |
| Security | Captures security posture, controls, threat model notes, vulnerability references, and assurance evidence. |
| KPIs | Records performance, fairness, reliability, safety, and operational measurements relevant to AI governance. |

### Format Discriminator

The schema requires `metadata.bomFormat` with the fixed value `AI-SBOM`. This lets validators, CLIs, APIs, and ingestion systems detect the document type without relying on file extensions or repository conventions.

### JSON Schema Draft 2020-12

The schema uses JSON Schema Draft 2020-12 and is intentionally strict. Modeled objects use `additionalProperties: false` so producers receive immediate feedback when they drift from the contract and consumers can reason about accepted documents predictably.

### JSON Signature Format

The optional `metadata.sbomAuthorSignature` field follows the JSON Signature Format `signaturecore` structure. JSF is already used by CycloneDX for enveloped JSON signatures, which makes it a practical fit for BOM-oriented tooling.

### JSON Canonicalization Scheme

The signing and verification flow uses RFC 8785 JSON Canonicalization Scheme semantics. Canonicalization prevents insignificant JSON formatting differences from changing the bytes that are signed or verified.

## Current Industry Challenges

AI-SBOM adoption is still emerging. Teams should expect technical and organizational friction.

| Challenge | Practical impact | Mitigation |
| --- | --- | --- |
| Fragmented ownership | Model, data, infrastructure, security, and release facts may be owned by different teams. | Define clear producer responsibilities and automate field population where possible. |
| Incomplete lifecycle data | Some fields are unknown at build time or only available after evaluation. | Use risk-tiered profiles and update AI-SBOMs at defined lifecycle gates. |
| Trust confusion | Teams may treat an embedded public key as proof of identity. | Separate cryptographic verification from trust-policy decisions. |
| Schema drift | Producers may add local fields that consumers cannot understand. | Use strict validation and plan versioned schema extensions. |
| Overloaded governance | Teams may try to encode every approval rule in the schema. | Keep schema validation separate from semantic policy checks. |
| Weak release integration | AI-SBOMs may be produced manually after deployment. | Generate, validate, sign, and verify in CI/CD and artifact workflows. |

These challenges are manageable if AI-SBOMs are introduced as part of a release and trust architecture rather than as static documentation.

## Technical Deep Dive

### Schema Identity

The immutable schema identifier is:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom-1.0.0.schema.json
```

A latest pointer is also published:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom.schema.json
```

Consumers that need reproducible validation should pin the immutable versioned URL. Consumers that intentionally want the newest compatible schema can use the latest pointer.

### Required Root Fields

The root object requires:

- `schemaVersion`
- `metadata`
- `system`
- `models`

The schema also supports optional `datasets`, `infrastructure`, `security`, and `kpis` sections. These sections are optional because lifecycle-dependent facts may become available at different stages of build, release, deployment, evaluation, or monitoring.

### Strict Validation

Strict validation is a design choice, not a convenience detail. It provides:

- Immediate producer feedback when the document does not match the contract.
- Predictable parsing for consumers and policy engines.
- More reliable automation in CI and verification workflows.
- Reduced ambiguity around hashes, identifiers, signatures, licenses, references, and relationship fields.

The tradeoff is that schema evolution must be deliberate. Producers cannot silently introduce new fields without a schema update or extension strategy.

### Author Signature Object

The supported signature object includes:

| Field | Required | Meaning |
| --- | --- | --- |
| `algorithm` | Yes | Signature algorithm label such as `ES256`, `RS256`, `PS256`, `Ed25519`, or a URI for proprietary algorithms. |
| `value` | Yes | Base64url signature value. This field is removed before canonicalization during signing and verification. |
| `keyId` | No | Key lookup identifier for a trust store, key management system, JWKS endpoint, or internal registry. |
| `publicKey` | No | Embedded public key material for self-contained cryptographic verification. |
| `certificatePath` | No | Certificate chain material for identity-binding workflows. |
| `excludes` | No | Optional JSF excludes. Verification policy should reject unexpected exclusions. |

The proof of concept supports simple JSF `signaturecore`. It does not currently support JSF `signers` multisignature objects or JSF `chain` signature-chain objects.

### Signed Payload

For embedded AI-SBOM signatures, the signed payload is the entire AI-SBOM JSON document after canonicalization, with only this field removed:

```text
metadata.sbomAuthorSignature.value
```

Other signature fields remain covered by the signature, including `algorithm`, `keyId`, `publicKey`, and `certificatePath`. This is important because the declared algorithm and key metadata become tamper-evident along with the business payload.

### Trust Boundary

Cryptographic verification proves that the signature matches the supplied public key. It does not prove that the key belongs to the claimed author or that the signer was authorized for the system.

Production verification needs a trust decision based on one or more of:

- `keyId`
- `certificatePath`
- A trusted key registry
- A key management system
- An organizational trust policy
- Out-of-band public key distribution

## Architecture / Process / Framework

The implementation turns the schema into an end-to-end trust workflow across three repositories:

- `shiftleftcyber/sbom-validator`: detects and validates AI-SBOM documents beside CycloneDX and SPDX.
- `shiftleftcyber/secure-sbom`: signs AI-SBOM documents through the v2 signing API and routes verification requests through the v2 verification API.
- `shiftleftcyber/securesbom-verifier`: verifies embedded AI-SBOM signatures in reusable library and offline CLI workflows.

### End-to-End Flow

| Step | Component | Technical action |
| --- | --- | --- |
| 1 | Producer | Build an AI-SBOM JSON document against `schemaVersion: 1.0.0`. |
| 2 | Validator | Detect AI-SBOM and validate against the embedded `ai-sbom-1.0.0` schema. |
| 3 | SecureSBOM sign API | Attach a JSF-style `metadata.sbomAuthorSignature` envelope without `value`. |
| 4 | Canonicalization | Canonicalize the whole document using RFC 8785 with only signature value omitted. |
| 5 | Signing backend | Sign `SHA-256(canonical bytes)` using the private key resolved from metadata. |
| 6 | Packaging | Embed the base64url signature value and return the signed AI-SBOM. |
| 7 | Verify API or offline verifier | Validate, recanonicalize the same signed surface, and verify with the public key. |
| 8 | Policy engine | Evaluate trust policy and semantic requirements for the workflow. |

### Validator Layer

The validator treats AI-SBOM as a first-class SBOM type beside CycloneDX and SPDX. It:

- Adds `AI-SBOM` as a supported SBOM type.
- Embeds the AI-SBOM schema under `schemas/ai-bom`.
- Detects AI-SBOM documents through `metadata.bomFormat == "AI-SBOM"`.
- Falls back to shape detection when `schemaVersion`, `metadata`, `system`, and `models` are present.
- Extracts `schemaVersion` as the AI-SBOM schema version.
- Validates the full document against the embedded AI-SBOM schema.
- Validates optional `metadata.sbomAuthorSignature` fields against the JSF `signaturecore` shape.

Example validation result:

```json
{
  "isValid": true,
  "sbomType": "AI-SBOM",
  "sbomVersion": "1.0.0",
  "detectedFormat": "JSON"
}
```

### Signing Layer

SecureSBOM signing is implemented in the v2 SBOM signing flow:

```text
POST /api/v2/sbom/sign
```

The signing service:

1. Validates the incoming SBOM with `sbom-validator`.
2. Resolves signing key metadata for the requested key.
3. Decodes the SBOM as a JSON object using number-preserving parsing.
4. Detects the normalized SBOM type.
5. For CycloneDX and AI-SBOM, creates a JSF-style signature envelope containing `algorithm`, `keyId`, and optional `publicKey`, but no `value`.
6. Attaches the envelope before signing.
7. Canonicalizes the full document with RFC 8785 JSON Canonicalization Scheme semantics.
8. Hashes the canonical bytes with SHA-256.
9. Uses the configured key backend to sign the hash.
10. Encodes the signature value as base64url without padding.
11. Embeds the signature value at `metadata.sbomAuthorSignature.value`.
12. Revalidates the signed document before returning it.

For AI-SBOM, embedded signing is the supported model. Detached AI-SBOM signatures are intentionally not supported in the v2 flow.

| Format | Signature model |
| --- | --- |
| CycloneDX | Embedded by default; detached supported when requested. |
| SPDX | Detached signature. |
| AI-SBOM | Embedded `metadata.sbomAuthorSignature`; detached not supported. |

### Verification Layer

The standalone verifier library is intentionally verification-only. It excludes key generation, private-key storage, signing, and key-store backends. That separation allows API services, offline CLIs, and other Go services to reuse the same verification logic.

AI-SBOM verification is exposed through:

```go
VerifyAIBOMEmbeddedVersioned(signedAIBOM, publicKeyPEM, VerificationV2)
VerifyAIBOMEmbeddedWithKeyVersioned(signedAIBOM, verificationKey, VerificationV2)
```

The verifier:

1. Validates the signed AI-SBOM with `sbom-validator`.
2. Extracts `metadata.sbomAuthorSignature`.
3. Reads the signature `value`.
4. Reads the declared `algorithm`, defaulting to `ES256` only for compatible legacy data.
5. Removes only the `value` field from the signature object.
6. Canonicalizes the entire AI-SBOM JSON document.
7. Verifies the signature with the supplied public key.
8. Returns a stable verification result.

Successful AI-SBOM verification returns:

```text
AI-BOM signature is valid
```

### SecureSBOM Verify API Routing

SecureSBOM v2 verification is routed by request shape:

```text
POST /api/v2/sbom/verify
```

Routing behavior:

- If `signature_b64` is present, the request is treated as SPDX detached verification.
- If the SBOM contains `metadata.bomFormat == "AI-SBOM"`, the request is routed to AI-SBOM embedded verification.
- Otherwise, the request is treated as CycloneDX embedded verification.

## Where SecureSBOM Fits

SecureSBOM provides the operational signing and verification layer for validated SBOM artifacts, including AI-SBOMs.

Its value-add is practical workflow integration:

- It validates AI-SBOMs before cryptographic operations are treated as meaningful.
- It signs canonicalized AI-SBOM metadata through the v2 signing API.
- It embeds signatures in the schema-defined `metadata.sbomAuthorSignature` location.
- It routes AI-SBOM verification through a dedicated embedded verification path.
- It supports reusable offline verification through `securesbom-verifier`.

SecureSBOM does not replace model registries, data catalogs, vulnerability scanners, policy engines, or governance workflows. It complements them by making AI-SBOM artifacts validatable, signable, distributable, and verifiable.

## Implementation Considerations

### Define Policy Profiles

Not every AI system needs the same level of metadata depth. Define profiles such as:

| Profile | Example systems | Suggested requirements |
| --- | --- | --- |
| Baseline | Internal prototypes and low-risk assistants | Required root fields, model identity, producer, intended use, basic security contacts. |
| Production | Customer-facing AI features | Dataset references, infrastructure, security controls, signed AI-SBOM, release verification. |
| Regulated | Healthcare, finance, safety, or legal workflows | Detailed provenance, evaluation metrics, approval evidence, certificate-backed signatures, retention rules. |
| High impact | Systems affecting rights, safety, or critical operations | Stronger evidence requirements, independent review, continuous monitoring, strict trust policy. |

### Automate Field Population

Manual AI-SBOM authoring should be limited. Prefer automated population from:

- Model registries.
- Dataset catalogs.
- CI build metadata.
- Git commits and release tags.
- Artifact stores.
- Infrastructure-as-code outputs.
- Security scanners and control inventories.
- Evaluation pipelines.

### Pin Schema Versions

Use immutable schema URLs for release gates and audit workflows. Use latest pointers only where teams intentionally accept schema updates.

### Separate Verification from Authorization

Signature verification should answer whether the document was changed after signing. Authorization should answer whether the signing key is trusted for that workflow.

### Make Offline Verification Possible

Offline verification is useful for regulated environments, incident response, customer evidence packages, and artifact validation where API access is unavailable or undesirable.

## Security, Compliance, and Operational Risks

| Risk | Description | Control |
| --- | --- | --- |
| False trust from embedded keys | An attacker can embed a public key that verifies their own signature. | Require trusted `keyId`, certificate chain, or registry lookup. |
| Weak canonicalization consistency | Signing and verifying different byte surfaces can cause false failures or false confidence. | Use RFC 8785 semantics consistently and test canonical fixtures. |
| Over-permissive schema handling | Accepting unknown fields can hide producer drift or malicious metadata. | Enforce strict schema validation. |
| Missing semantic checks | A valid signed document may still omit required organizational evidence. | Run policy checks after schema and signature verification. |
| Key lifecycle gaps | Old, compromised, or unauthorized keys may remain trusted. | Define issuance, rotation, revocation, and audit procedures. |
| Incomplete AI lifecycle coverage | Build-time documents may miss deployment or monitoring facts. | Update AI-SBOMs at defined lifecycle gates. |
| Misaligned ownership | Teams may disagree over who produces or signs metadata. | Assign producer, reviewer, signer, and verifier roles. |

Compliance teams should treat signed AI-SBOMs as structured evidence, not as a complete compliance program. They can support auditability, traceability, and repeatability, but still require policy interpretation and governance decisions.

## Evaluation Criteria

Organizations evaluating AI-SBOM tooling should assess:

| Criterion | Questions to ask |
| --- | --- |
| Schema support | Does the tool validate AI-SBOM `1.0.0` strictly and expose useful errors? |
| Format detection | Can it reliably distinguish AI-SBOM from CycloneDX, SPDX, and arbitrary JSON? |
| Signature semantics | Does it sign and verify the same canonicalized payload? |
| Key handling | Does it support key identifiers, certificate material, or trusted key registries? |
| Offline operation | Can verification run without a live API dependency? |
| CI/CD integration | Can validation and verification run in build, release, and deployment gates? |
| Policy integration | Can semantic policies run after schema and signature verification? |
| Versioning | Can producers and consumers pin schema versions? |
| Evidence export | Can signed AI-SBOMs be retained, distributed, and audited? |

## Recommended Approach

Adopt AI-SBOMs incrementally.

1. Inventory production AI systems and classify them by risk tier.
2. Generate baseline AI-SBOMs for representative systems.
3. Validate documents against the strict `1.0.0` schema.
4. Add CI checks that block malformed AI-SBOMs.
5. Define signing authority and key-management policy.
6. Sign AI-SBOMs at release or publication time.
7. Verify signed AI-SBOMs before deployment, distribution, procurement approval, or audit acceptance.
8. Add semantic policy checks for required fields, trusted signers, risk-tier evidence, and lifecycle completeness.
9. Expand coverage to datasets, infrastructure, security evidence, and KPIs.
10. Review schema evolution needs after real producer and consumer feedback.

This staged approach keeps early adoption practical while creating a path toward stronger assurance.

## Example Use Cases

### Release Gate for a Customer-Facing AI Feature

A platform team requires every production AI release to include a valid signed AI-SBOM. CI validates the document, SecureSBOM signs it, and the deployment pipeline verifies the signature before promotion. A policy check ensures the model version, dataset references, and security contact fields are present.

### Procurement Intake for a Third-Party AI Service

A security review team requests an AI-SBOM from a vendor. The team validates schema conformance, checks that the document is signed by an approved vendor key, and evaluates whether the declared model, dataset, infrastructure, and security properties meet intake requirements.

### Offline Audit Evidence

A regulated team exports signed AI-SBOMs with release artifacts. Auditors can verify the signatures offline and compare the structured metadata against documented approval records.

### Incident Response

When a dataset, model family, endpoint, or vulnerability becomes relevant to an incident, responders query AI-SBOM inventory records to identify affected systems and verify that evidence artifacts have not been altered.

## Common Pitfalls

- Treating an AI-SBOM as a one-time document instead of a lifecycle artifact.
- Signing invalid or incomplete metadata before schema validation.
- Assuming a successful signature check proves signer authorization.
- Letting producers add local fields without schema governance.
- Using latest schema URLs in workflows that require reproducibility.
- Omitting policy checks because schema validation succeeded.
- Failing to test canonicalization across implementations and key backends.
- Asking teams to hand-write metadata that can be generated from existing systems.

## Future Outlook

AI-SBOM practices are likely to mature in several directions:

- Richer schema versions for evaluation evidence, attestations, policy decisions, and lifecycle events.
- Multisignature and signature-chain support for workflows involving multiple accountable parties.
- Stronger integration with model registries, data catalogs, artifact stores, and policy engines.
- More precise trust profiles for regulated and high-impact AI systems.
- Customer-facing evidence packages that combine signed AI-SBOMs with vulnerability, model, and control attestations.
- Automated drift detection between declared AI-SBOM metadata and runtime reality.

The immediate opportunity is narrower and practical: make AI supply chain metadata structured, validatable, signed, and verifiable before organizations try to automate higher-order governance decisions.

## Conclusion

AI supply chain assurance requires more than a conventional package inventory. Teams need structured metadata about models, datasets, infrastructure, security controls, and operational indicators. They also need confidence that the metadata they receive has not been modified after publication.

The ShiftLeftCyber AI-BOM proof of concept provides a strict AI-SBOM schema based on recognized minimum-element clusters. SecureSBOM extends that schema into an operational workflow for signing and verifying AI-SBOM artifacts. Together, they establish a practical foundation for AI inventory, release governance, procurement review, offline verification, and policy automation.

The central design principle is separation of concerns: validate structure with the schema, protect integrity with signatures, decide trust through key policy, and evaluate business or compliance requirements through semantic policy.

## Call to Action

Request access to the full SecureSBOM AI-SBOM implementation package to review example schemas, signed fixtures, verification workflows, CI integration patterns, and policy templates for production AI supply chain assurance.

## References

- [AI-SBOM schema v1.0.0](../docs/schemas/ai-sbom-1.0.0.schema.json)
- [AI-SBOM latest schema](../docs/schemas/ai-sbom.schema.json)
- [SBOM for AI minimum elements PDF](../docs/assets/SBOM-for-AI_minimum-elements.pdf)
- `shiftleftcyber/sbom-validator`
- `shiftleftcyber/secure-sbom`
- `shiftleftcyber/securesbom-verifier`
