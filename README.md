# AI SBOM Schema and Examples

This repository contains an AI SBOM JSON Schema derived from the minimum element clusters in `SBOM-for-AI_minimum-elements.pdf`.

Published docs and schema URLs are available at:

[https://shiftleftcyber.io/ai-bom/](https://shiftleftcyber.io/ai-bom/)

The source PDF is also published for direct download at:

[https://shiftleftcyber.io/ai-bom/assets/SBOM-for-AI_minimum-elements.pdf](https://shiftleftcyber.io/ai-bom/assets/SBOM-for-AI_minimum-elements.pdf)

## Files

- `ai-sbom.schema.json`: Draft 2020-12 JSON Schema for an AI SBOM.
- `docs/schemas/ai-sbom-1.0.0.schema.json`: Immutable versioned schema URL for GitHub Pages.
- `docs/schemas/ai-sbom.schema.json`: Latest schema URL for GitHub Pages.
- `docs/assets/SBOM-for-AI_minimum-elements.pdf`: GitHub Pages-hosted copy of the source PDF.
- `examples/valid/customer-support-ai-sbom.json`: Valid SBOM for a customer support assistant.
- `examples/valid/medical-triage-ai-sbom.json`: Valid SBOM for a medical triage recommender.
- `examples/invalid/missing-required-metadata.json`: Invalid because required metadata and required nested cluster fields are missing. It intentionally omits `sbomTimestamp`; `sbomAuthorSignature` is optional.
- `examples/invalid/bad-types-and-enums.json`: Invalid because several fields use the wrong type, invalid enum values, or empty arrays where at least one item is required.
- `examples/invalid/non-jsf-signature.json`: Invalid because `metadata.sbomAuthorSignature` does not follow the JSF signaturecore structure.
- `examples/invalid/unknown-extra-properties.json`: Invalid because extra properties are disallowed and the model hash algorithm/value are invalid.

## Source Mapping

The schema models the seven clusters described in the PDF:

- Metadata
- System Level Properties
- Models
- Dataset Properties
- Infrastructure
- Security Properties
- Key Performance Indicators

For automation, the schema requires `metadata.bomFormat` with the fixed value `AI-SBOM`. This gives tools a small discriminator field for identifying this format without relying on file names.

## Author Signatures

The optional `metadata.sbomAuthorSignature` field uses the JSON Signature Format (JSF) `signaturecore` structure. JSF is also used by CycloneDX for enveloped JSON signatures, so adopting it gives AI SBOM producers and consumers a familiar signing model instead of creating a new signature format.

This schema currently supports only the simple JSF `signaturecore` form. It does not yet support JSF `signers` multisignature or `chain` signature-chain objects. The core fields are:

- `algorithm`: JSF/JWA signature algorithm such as `ES256`, `RS256`, `PS256`, `Ed25519`, or a URI for proprietary algorithms.
- `value`: the base64url-encoded signature value.
- `keyId`: optional key identifier for lookup in a trust store, key management system, JWKS endpoint, or internal registry.
- `publicKey`: optional embedded public key that enables self-contained cryptographic verification.
- `certificatePath`: optional certificate chain material for workflows that need identity binding through X.509 trust anchors.

The signature follows JSF signing semantics. For a simple signature, the signed payload is the entire AI SBOM JSON document after JSON Canonicalization Scheme processing, with only `metadata.sbomAuthorSignature.value` removed before canonicalization. Other signature fields, including `algorithm`, `keyId`, `publicKey`, and `certificatePath`, remain part of the signed payload.

Embedding a `publicKey` makes cryptographic verification easier, but it does not by itself prove that the key belongs to the claimed SBOM author. Verifiers still need a trust decision based on `keyId`, `certificatePath`, an out-of-band trust store, or an organizational policy. If JSF `excludes` is used, verifiers should reject unexpected exclusions by policy.

## Consuming the Schema

Use the immutable versioned URL when you want reproducible validation:

[https://shiftleftcyber.io/ai-bom/schemas/ai-sbom-1.0.0.schema.json](https://shiftleftcyber.io/ai-bom/schemas/ai-sbom-1.0.0.schema.json)

Use the latest URL when you intentionally want the newest compatible schema:

[https://shiftleftcyber.io/ai-bom/schemas/ai-sbom.schema.json](https://shiftleftcyber.io/ai-bom/schemas/ai-sbom.schema.json)

You can also pin directly to a Git tag:

[https://raw.githubusercontent.com/shiftleftcyber/ai-bom/v1.0.0/ai-sbom.schema.json](https://raw.githubusercontent.com/shiftleftcyber/ai-bom/v1.0.0/ai-sbom.schema.json)

The schema is intentionally strict with `additionalProperties: false` so that nonconforming examples fail predictably, but lifecycle-dependent fields such as signatures, hashes, licenses, security evidence, infrastructure details, and KPIs are optional.

## License

This project is licensed under the [Apache License 2.0](LICENSE). Apache-2.0 was chosen because it is permissive like MIT while also including an explicit patent grant, which is useful for an interoperability-focused schema and related tooling.
