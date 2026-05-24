# AI SBOM Schema and Examples

This repository contains an AI SBOM JSON Schema derived from the minimum element clusters in `SBOM-for-AI_minimum-elements.pdf`.

The source PDF is also published for direct download at:

```text
https://shiftleftcyber.io/ai-bom/assets/SBOM-for-AI_minimum-elements.pdf
```

## Files

- `ai-sbom.schema.json`: Draft 2020-12 JSON Schema for an AI SBOM.
- `docs/schemas/ai-sbom-1.0.0.schema.json`: Immutable versioned schema URL for GitHub Pages.
- `docs/schemas/ai-sbom.schema.json`: Latest schema URL for GitHub Pages.
- `docs/assets/SBOM-for-AI_minimum-elements.pdf`: GitHub Pages-hosted copy of the source PDF.
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

For automation, the schema requires `metadata.bomFormat` with the fixed value `AI-SBOM`. This gives tools a small discriminator field for identifying this format without relying on file names.

## Consuming the Schema

Use the immutable versioned URL when you want reproducible validation:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom-1.0.0.schema.json
```

Use the latest URL when you intentionally want the newest compatible schema:

```text
https://shiftleftcyber.io/ai-bom/schemas/ai-sbom.schema.json
```

You can also pin directly to a Git tag:

```text
https://raw.githubusercontent.com/shiftleftcyber/ai-bom/v1.0.0/ai-sbom.schema.json
```

The schema is intentionally strict with `additionalProperties: false` so that nonconforming examples fail predictably, but lifecycle-dependent fields such as signatures, hashes, licenses, security evidence, infrastructure details, and KPIs are optional.
