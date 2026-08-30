# Protein Ensemble (PE) Manifest — Data Contract Reference [v0.1.0]

Plain-English companion to `manifest.schema.json`. This document defines the **Protein Ensemble manifest data contract and semantic requirements**.

A PE package consists of one `manifest.yaml` plus the resources it references, organized within a directory tree.

## Serialization and Conformance

PE manifests are serialized as **YAML 1.2**.

The normative structural contract is defined by:

`manifest.schema.json`

The YAML representation is restricted to the data model expressible by that schema.

PE manifests:

- MUST use YAML 1.2.
- MUST conform to the corresponding `manifest.schema.json`.
- MUST NOT use custom YAML tags.
- MUST NOT use YAML aliases or anchors.
- MUST NOT contain duplicate mapping keys.
- MUST use only data types representable by the JSON data model defined by the schema.

The terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

JSON MAY be used as an interchange representation of the same manifest data model, but `manifest.yaml` is the canonical package manifest filename.

## Package Structure

A minimal PE package has the following form:

```text
ensemble_001/
├── manifest.yaml
└── structures/
    ├── state_001.cif
    ├── state_002.cif
    └── state_003.cif
```

The manifest identifies the resources required to interpret the ensemble.

Relative resource URIs are resolved relative to the PE package root, it MUST NOT traverse outside the PE package root [e.g., it MUST NOT contain ../ segments that resolve to a parent directory].

## Manifest

`manifest.yaml` has the following top-level fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schemaVersion` | string (`"0.1.0"`) | Yes | Version of the Protein Ensemble manifest specification. |
| `id` | string | Yes | Identifier for the ensemble within its managing namespace. |
| `contentHash` | string (`"algo:hexdigest"`) | Yes | Content identifier for the ensemble, in the form algorithm:hexdigest. |
| `weightScheme` | object | No | Defines how member weights are interpreted. |
| `capabilitiesRequired` | array of strings | No | Capabilities a consumer must support to interpret this manifest. |
| `metadata` | object | No | Non-normative metadata; fields have no PE core semantic meaning unless explicitly defined by the specification. |
| `members` | object | Yes | Mapping of member structures. |
  
### Terminology

A **member** is a structure-bearing unit included in the ensemble.

A member MAY represent an individual conformer, a trajectory frame, a representative structure, or another structure representation permitted by this contract.

## Member

Each key in the `members` object represents one ensemble member ID.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `structure` | object | Yes | Defines the structural resource |
| `structureHash` | string | Yes | Content identifier for the referenced structural resource, in the form algorithm:hexdigest. |
| `weight` | object | No | Contains the member weight. |

### Member IDs

Member `id` values MUST be unique within the ensemble.

To ensure cross-platform compatibility and safe URI/filesystem mapping, id strings MUST contain only alphanumeric characters, hyphens, and underscores (matching the regular expression `^[a-zA-Z0-9_-]+$`).

## Structure

A member's `structure` object contains the following properties:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `uri` | string (uri-reference) | Yes | URI reference to the structural resource. Relative references are resolved relative to the manifest. |
| `modelIndex` | integer (>= 0) | No | Zero-based model index within the referenced structural resource. When omitted, the referenced resource represents a single structural member. |

## Weighting

`weightScheme` is required when any member declares a `weight`.

The `weightScheme.type` MUST be one of:

| Type | Description |
| --- | --- |
| `EQUILIBRIUM_PROBABILITY` | Equilibrium probability weighting. |
| `UNIFORM` | Uniform weighting across members. |

`weightScheme` also includes a `normalized` property (boolean) that dictates whether explicit member weights are normalized according to the semantic contract.

### Weighting rules

- `weightScheme` MUST be present if any member has a `weight`.
- If `weightScheme.normalized == true`, all member weights MUST sum to `1.0` within a tolerance of `1e-6`.
- For `UNIFORM`, all members are understood to have equal weight. If explicit member weights are provided, they MUST be consistent with uniform weighting.
- A member's `weight` object contains a single `value` (number) representing its weight; its interpretation and permitted range are defined by `weightScheme` and the semantic contract.

## Capabilities

`capabilitiesRequired` declares capabilities that a consumer must support to interpret the package. If omitted, consumers should rely on the default standalone CIF reading capabilities.

## Opaque Fields

Fields explicitly described as **opaque passthrough** (like `metadata`) are intentionally not interpreted by the PE core contract.

Their contents MAY be defined by extensions or downstream applications.

The PE core contract does not assign domain-specific semantics to these fields.

Whether unknown fields are permitted outside explicitly extensible objects is determined by `manifest.schema.json`.

## Semantic Constraints

The following requirements are semantic and may not be fully expressible through JSON Schema alone.

A manifest can therefore satisfy the structural schema while still violating the PE contract.

A conforming package MUST satisfy all of the following:

- Member `id` values MUST be unique within the ensemble.
- `weightScheme` presence or absence MUST agree with whether any member declares a `weight`.
- If `weightScheme.normalized == true`, member weights MUST sum to `1.0` within a tolerance of `1e-6`.
- `contentHash` MUST match the content hash computed for the package according to the PE content-hash specification.

## Content Integrity

`contentHash` has the form:

```text
algorithm:hexdigest
```

For example:

```text
blake3:ab12...
```

The value identifies a content hash computed over the package according to the applicable PE content-hash specification.

A syntactically valid hash MUST NOT be treated as evidence that the package contents are correct.

An implementation performing integrity verification MUST recompute the hash from the package contents and compare the result with the manifest value.

The content-hash algorithm, Merkle-tree construction, canonical serialization rules, and resource-byte extraction rules are **not defined by this document**. They are defined by the separate PE content-hash specification.

The content hash identifies the package's content rather than the particular YAML formatting used to serialize its manifest.

Equivalent YAML and JSON representations of the same PE data model MUST therefore be capable of representing the same package content without changing its content identity.

## Versioning

`schemaVersion` uses the form:

```text
X.Y.Z
```

The currently defined version is:

```text
0.1.0
```

Changes to the manifest contract MUST be reflected in the schema version according to the PE versioning policy.

The precise compatibility guarantees between different schema versions are outside the scope of this document.

## Reproducibility and Scope

The PE core manifest describes **what an ensemble contains and how its structural contents are interpreted**.

It does not attempt to fully describe **how the ensemble was generated**.

Reproducibility of the package as a data artifact depends on:

- stable identification of the package;
- identification of its structural resources;
- unambiguous interpretation of those resources;
- and verification of `contentHash`.

Workflow provenance, generation parameters, software environments, random seeds, simulation workflows, and derivation histories are outside the scope of the PE core manifest.

Such information MAY be represented by separate PE extensions or higher-level workflow/provenance systems.

## Out of Scope

This document defines the data contract, but intentionally excludes the following:

- **Implementation Details:** Tooling for YAML parsing or schema validation.
- **Hashing Mechanics:** BLAKE3 Merkle-tree construction, canonical serialization rules, and trajectory-to-byte extraction methods.
- **Scientific Semantics:** The biochemical interpretation of members as conformational states, the algorithms used to calculate weights, and application-specific logic for opaque objects (`metadata`, `dynamics`, `thermodynamics`).
- **Provenance:** Workflow lineage, generation parameters, and external catalog systems that link an ensemble to its derivation history.
