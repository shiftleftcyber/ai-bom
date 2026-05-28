# AI-BOM Schema PoC and SecureSBOM Signing and Verification

Technical whitepaper  
ShiftLeftCyber AI-BOM proof of concept  
May 27, 2026

## Executive Summary

This whitepaper documents the AI-BOM schema proof of concept and the SecureSBOM implementation work that made AI-SBOM signing and verification operational. The PoC defines a machine-validatable JSON Schema for AI Bills of Materials, grounded in the minimum-element clusters from the G7 SBOM for AI work: metadata, system-level properties, models, dataset properties, infrastructure, security properties, and key performance indicators.

SecureSBOM was extended across three repositories:

- [`shiftleftcyber/sbom-validator`](https://github.com/shiftleftcyber/sbom-validator): detects and validates AI-SBOM documents beside CycloneDX and SPDX.
- [`shiftleftcyber/secure-sbom`](https://github.com/shiftleftcyber/secure-sbom): signs AI-SBOM documents through the v2 signing API and routes verification requests through the v2 verification API.
- [`shiftleftcyber/securesbom-verifier`](https://github.com/shiftleftcyber/securesbom-verifier): verifies embedded AI-SBOM signatures in reusable library and offline CLI workflows.

The result is an end-to-end trust workflow for AI supply chain metadata: create an AI-SBOM, validate it against the schema, sign the canonicalized document, distribute the signed artifact, and verify its integrity before relying on it.

## Problem Context

Traditional SBOM formats are strong at software component inventory, but AI systems introduce additional supply chain facts: model identity, model lineage, training and fine-tuning datasets, data sensitivity, inference infrastructure, model behavior limits, security evidence, and operational KPIs. These facts matter because model behavior and deployment risk are shaped by more than package dependencies.

The PoC addresses that gap with a JSON-native AI-SBOM format that can be validated, version-pinned, signed, and consumed by automated systems. It avoids a loose documentation-only model by treating AI inventory data as a strict schema surface.

## Schema Design

The schema is published as version `1.0.0` and uses JSON Schema Draft 2020-12. The immutable schema identifier is:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom-1.0.0.schema.json
```

A latest pointer is also available:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom.schema.json
```

The root object requires:

- `schemaVersion`
- `metadata`
- `system`
- `models`

The schema also supports optional `datasets`, `infrastructure`, `security`, and `kpis` sections. These sections remain optional because some AI lifecycle facts are not always available at the same point in a build, release, or governance workflow.

### Minimum-Element Clusters

| Cluster | Purpose |
| --- | --- |
| Metadata | Names the format, producer, version, timestamp, tool context, dependency relationships, and optional author signature. |
| System | Captures the AI system name, components, producer, version, data flow, input and output properties, and intended application area. |
| Models | Records model names, identifiers, versions, producers, hashes, architecture, training properties, licenses, and references. |
| Datasets | Describes dataset identity, provenance, content, hashes, sensitivity, licenses, and dataset dependency relationships. |
| Infrastructure | Documents runtime software, hardware, deployment environment, endpoints, and operational dependencies. |
| Security | Captures security posture, controls, threat model notes, vulnerability references, and assurance evidence. |
| KPIs | Records performance, fairness, reliability, safety, and operational measurements relevant to AI governance. |

### Format Discriminator

The schema requires `metadata.bomFormat` with the fixed value:

```json
"AI-SBOM"
```

This gives validators, CLIs, APIs, and ingestion systems a small discriminator field for routing AI-SBOM documents without relying on filenames or repository-specific conventions.

### Strict Validation

The schema uses `additionalProperties: false` throughout the modeled objects. That strictness is deliberate:

- Producers get immediate feedback when they drift from the contract.
- Consumers can trust that accepted documents match the known data model.
- Invalid examples fail predictably during automation.
- Security-sensitive fields such as hashes, signatures, licenses, and references are checked for shape rather than accepted as arbitrary JSON.

## Author Signature Model

The optional `metadata.sbomAuthorSignature` field follows the JSON Signature Format (JSF) `signaturecore` structure. JSF is already used by CycloneDX for enveloped JSON signatures, so the AI-SBOM PoC follows an existing BOM-oriented signing model rather than inventing a new signature format.

The supported signature object includes:

| Field | Meaning |
| --- | --- |
| `algorithm` | Required algorithm label such as `ES256`, `RS256`, `PS256`, `Ed25519`, or a URI for proprietary algorithms. |
| `value` | Required base64url signature value. This field is removed before canonicalization during signing and verification. |
| `keyId` | Optional key lookup identifier for a trust store, key management system, JWKS endpoint, or internal registry. |
| `publicKey` | Optional embedded public key material for self-contained cryptographic verification. |
| `certificatePath` | Optional certificate chain material for identity binding workflows. |
| `excludes` | Optional JSF excludes. Verification policy should reject unexpected exclusions. |

The PoC currently supports the simple JSF `signaturecore` form. It does not yet support JSF `signers` multisignature objects or JSF `chain` signature-chain objects.

### Signed Payload

For embedded AI-SBOM signatures, the signed payload is the entire AI-SBOM JSON document after JSON Canonicalization Scheme processing, with only this field removed:

```text
metadata.sbomAuthorSignature.value
```

Other signature fields remain covered by the signature, including:

- `algorithm`
- `keyId`
- `publicKey`
- `certificatePath`

This matters because the signature metadata itself becomes tamper-evident. A verifier is not merely checking the business payload; it is also checking the declared algorithm and key metadata that were present when the signature was produced.

### Trust Boundary

An embedded `publicKey` can make cryptographic verification easier, but it does not by itself prove that the key belongs to the claimed SBOM author. A production verifier still needs a trust decision based on one or more of:

- `keyId`
- `certificatePath`
- a trusted key registry
- a key management system
- an organizational trust policy
- an out-of-band public key distribution mechanism

## SecureSBOM Implementation

The implementation turns the schema PoC into a usable signing and verification workflow.

## Validator Layer: `sbom-validator`

The validator now treats AI-SBOM as a first-class SBOM type beside CycloneDX and SPDX.

Key implementation behavior:

- Adds `AI-SBOM` as a supported SBOM type.
- Embeds the AI-SBOM schema under `schemas/ai-bom`.
- Detects AI-SBOM documents through `metadata.bomFormat == "AI-SBOM"`.
- Falls back to AI-SBOM shape detection when `schemaVersion`, `metadata`, `system`, and `models` are present.
- Extracts `schemaVersion` as the AI-SBOM schema version.
- Validates the full document against the embedded AI-SBOM schema.
- Validates the optional `metadata.sbomAuthorSignature` field against the JSF `signaturecore` shape.

This layer is important because both signing and verification begin with schema validation. Invalid documents are rejected before any cryptographic operation is treated as meaningful.

Example validation result:

```json
{
  "isValid": true,
  "sbomType": "AI-SBOM",
  "sbomVersion": "1.0.0",
  "detectedFormat": "JSON"
}
```

## Signing Layer: `secure-sbom`

SecureSBOM signing is implemented in the v2 SBOM signing flow. The v2 endpoint accepts a wrapped request containing a `key_id`, the raw SBOM JSON, and formatting options.

```text
POST /api/v2/sbom/sign
```

The signing service performs this flow:

1. Validate the incoming SBOM with `sbom-validator`.
2. Resolve signing key metadata for the requested key.
3. Decode the SBOM as a JSON object using number-preserving parsing.
4. Detect the normalized SBOM type.
5. For CycloneDX and AI-SBOM, create a JSF-style signature envelope containing `algorithm`, `keyId`, and optional `publicKey`, but no `value`.
6. Attach the envelope before signing.
7. Canonicalize the full document with RFC 8785 JSON Canonicalization Scheme.
8. Hash the canonical bytes with SHA-256.
9. Ask the configured key backend to sign the hash.
10. Encode the resulting AI-SBOM signature as base64url without padding.
11. Embed the signature value at `metadata.sbomAuthorSignature.value`.
12. Revalidate the signed document before returning it.

For AI-SBOM, the embedded signature location is:

```text
metadata.sbomAuthorSignature
```

The returned AI-SBOM contains a signature object shaped like:

```json
{
  "metadata": {
    "bomFormat": "AI-SBOM",
    "sbomAuthorSignature": {
      "algorithm": "ES256",
      "keyId": "production-ai-bom-key-2026-05",
      "publicKey": {
        "kty": "EC",
        "crv": "P-256",
        "x": "...",
        "y": "..."
      },
      "value": "..."
    }
  }
}
```

### Detached Signature Behavior

SecureSBOM intentionally does not support detached AI-SBOM signatures in the v2 flow. If a caller requests detached signing for AI-SBOM, the service logs a structured warning and returns an embedded signature.

The supported signing behaviors are:

| Format | Signature model |
| --- | --- |
| CycloneDX | Embedded by default; detached supported when requested. |
| SPDX | Detached signature. |
| AI-SBOM | Embedded `metadata.sbomAuthorSignature`; detached not supported. |

## Verification Layer: `securesbom-verifier`

The standalone verifier library is intentionally verification-only. It excludes key generation, private-key storage, signing, and key store backends. That separation allows API services, offline CLIs, and other Go services to reuse the same verification logic.

The AI-SBOM verification API is exposed through:

```go
VerifyAIBOMEmbeddedVersioned(signedAIBOM, publicKeyPEM, VerificationV2)
VerifyAIBOMEmbeddedWithKeyVersioned(signedAIBOM, verificationKey, VerificationV2)
```

AI-SBOM verification is only available with `VerificationV2`.

The verifier performs this flow:

1. Validate the signed AI-SBOM with `sbom-validator`.
2. Extract `metadata.sbomAuthorSignature`.
3. Read the signature `value`.
4. Read the declared `algorithm`; default to `ES256` if compatible legacy data omits it.
5. Remove only the `value` field from the signature object.
6. Canonicalize the entire AI-SBOM JSON document using RFC 8785.
7. Verify the signature with the supplied public key.
8. Return a stable verification result.

Successful AI-SBOM verification returns the message:

```text
AI-BOM signature is valid
```

## SecureSBOM Verify API Routing

The SecureSBOM v2 verify handler routes requests by request shape:

```text
POST /api/v2/sbom/verify
```

Routing behavior:

- If `signature_b64` is present, the request is treated as SPDX detached verification.
- If the SBOM contains `metadata.bomFormat == "AI-SBOM"`, the request is routed to AI-SBOM embedded verification.
- Otherwise, the request is treated as CycloneDX embedded verification.

The verify service resolves the public key from stored key metadata, constructs a `VerificationKey`, and delegates cryptographic verification to `securesbom-verifier`.

## End-to-End Flow

| Step | Component | Technical action |
| --- | --- | --- |
| 1 | Producer | Build an AI-SBOM JSON document against `schemaVersion: 1.0.0`. |
| 2 | `sbom-validator` | Detect AI-SBOM and validate against the embedded `ai-sbom-1.0.0` schema. |
| 3 | SecureSBOM sign API | Attach a JSF-style `metadata.sbomAuthorSignature` envelope without `value`. |
| 4 | Canonicalization | Canonicalize the entire document using RFC 8785 with only signature value omitted. |
| 5 | Signing backend | Sign `SHA-256(canonical bytes)` using the private key resolved from metadata. |
| 6 | Packaging | Embed base64url signature value and return the signed AI-SBOM. |
| 7 | SecureSBOM verify API or offline verifier | Validate, recanonicalize the same signed surface, and verify with the public key. |

## Security Properties

The implementation provides several useful security properties:

- **Integrity:** Any change to covered AI-SBOM content changes the canonical bytes and invalidates the signature.
- **Format safety:** Strict schema validation blocks malformed AI-SBOMs before signing or verification succeeds.
- **Algorithm agility:** The schema allows common JWA algorithm labels and URI-based proprietary algorithms, while implementation can restrict what each service supports.
- **Trust separation:** Cryptographic verification proves possession of the private key, while identity trust is handled through key metadata, public key registries, or certificates.
- **Offline verification:** `securesbom-verifier` and its optional CLI allow signed AI-SBOMs to be checked without a live SecureSBOM API dependency.
- **Interoperability:** Reusing JSF concepts keeps the AI-SBOM signature model close to CycloneDX embedded signatures.

## Current Limitations

The current PoC and SecureSBOM implementation intentionally keep scope tight:

- The schema supports a single JSF `signaturecore` object, not JSF multisignature or chain signatures.
- AI-SBOM signing and verification are v2-only in SecureSBOM because they depend on RFC 8785 canonicalization semantics.
- Detached AI-SBOM signatures are not supported yet.
- Embedded `publicKey` material is not a trust decision by itself.
- Verification focuses on schema validity and signature validity; semantic policy checks should be layered above schema validation.
- The schema is strict, which is good for interoperability but requires producers to update when new fields are introduced in future schema versions.

## Recommended Next Steps

1. Define AI-SBOM policy profiles for internal, customer-facing, regulated, and high-impact AI systems.
2. Add conformance tests that sign and verify canonical fixtures across Go versions and key backends.
3. Define trust policy for `keyId`, `publicKey`, and `certificatePath`.
4. Consider future schema versions for multisignature, attestations, policy decisions, and richer model-evaluation evidence.
5. Publish producer guidance that explains canonicalization, required clusters, optional lifecycle fields, and signature handling.
6. Add CI examples that validate AI-SBOM documents and verify signed AI-SBOM artifacts before release.

## Implementation Evidence

This paper is based on the schema PoC and local implementation evidence from:

- `ai_bom_poc/README.md`
- `ai_bom_poc/docs/schemas/ai-sbom-1.0.0.schema.json`
- `ai_bom_poc/examples/valid/customer-support-ai-sbom.json`
- `shiftleftcyber/sbom-validator` AI-SBOM detection and embedded schema support
- `secure-sbom` v2 signing and verification handlers
- `secure-sbom` AI-SBOM signature attachment and RFC 8785 canonicalization helpers
- `secure-sbom` signing backend abstraction for file and GCP KMS keys
- `securesbom-verifier` AI-SBOM verification APIs and offline verification CLI

## References

- [AI-SBOM schema v1.0.0](./schemas/ai-sbom-1.0.0.schema.json)
- [AI-SBOM latest schema](./schemas/ai-sbom.schema.json)
- [SBOM for AI minimum elements PDF](./assets/SBOM-for-AI_minimum-elements.pdf)
- [shiftleftcyber/sbom-validator](https://github.com/shiftleftcyber/sbom-validator)
- [shiftleftcyber/secure-sbom](https://github.com/shiftleftcyber/secure-sbom)
- [shiftleftcyber/securesbom-verifier](https://github.com/shiftleftcyber/securesbom-verifier)
