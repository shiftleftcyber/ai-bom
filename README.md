# AI SBOM Schema and Examples

This repository contains an AI SBOM JSON Schema derived from the minimum element clusters in `SBOM-for-AI_minimum-elements.pdf`.

## Files

- `ai-sbom.schema.json`: Draft 2020-12 JSON Schema for an AI SBOM.
- `examples/valid/customer-support-ai-sbom.json`: Valid SBOM for a customer support assistant.
- `examples/valid/medical-triage-ai-sbom.json`: Valid SBOM for a medical triage recommender.
- `examples/invalid/missing-required-metadata.json`: Invalid because required metadata and required nested cluster fields are missing. It intentionally omits `sbomTimestamp`; `sbomAuthorSignature` is optional.
- `examples/invalid/bad-types-and-enums.json`: Invalid because several fields use the wrong type, invalid enum values, or empty arrays where at least one item is required.
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

The schema is intentionally strict with `additionalProperties: false` so that nonconforming examples fail predictably, but lifecycle-dependent fields such as signatures, hashes, licenses, security evidence, infrastructure details, and KPIs are optional.
