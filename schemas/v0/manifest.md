# PCE Manifest — Data Contract Reference (v0.1.0)

Plain-English companion to `manifest.schema.json`. This document defines the **PCE manifest data contract and semantic requirements**. It intentionally does not define implementation details such as parsing, validation code, or content-hash algorithms.

A PCE package consists of one `manifest.yaml` plus the resources it references, organized within a directory tree.

## Serialization and Conformance

PCE manifests are serialized as **YAML 1.2**.

The normative structural contract is defined by:

```text
manifest.schema.json
```

The YAML representation is restricted to the data model expressible by that schema.

PCE manifests:

- MUST use YAML 1.2.
- MUST conform to the corresponding `manifest.schema.json`.
- MUST NOT use custom YAML tags.
- MUST NOT use YAML aliases or anchors.
- MUST NOT contain duplicate mapping keys.
- MUST use only data types representable by the JSON data model defined by the schema.

The terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

JSON MAY be used as an interchange representation of the same manifest data model, but `manifest.yaml` is the canonical package manifest filename.

## Package Structure

A minimal PCE package has the following form:

```text
ensemble_001/
├── manifest.yaml
└── structures/
    ├── state_001.cif
    ├── state_002.cif
    └── state_003.cif
```

The manifest identifies the resources required to interpret the ensemble.

Relative resource URIs are resolved relative to the PCE package root. If a URI is a relative path, it MUST NOT traverse outside the PCE package root (e.g., it MUST NOT contain ../ segments that resolve to a parent directory).

The exact rules governing external URI schemes and external resources are outside the scope of this document.

## Manifest

`manifest.yaml` has the following top-level fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string (`"X.Y.Z"`) | Yes | Currently `"1.0.0"`. |
| `id` | string | Yes | Ensemble ID. |
| `content_hash` | string (`"algo:hexdigest"`) | Yes | Content hash of the package according to the PCE content-hash specification. |
| `topology_reference` | object | Yes | Exactly one of `member_id` or `external_reference`. |
| `weight_scheme` | object | Only if any member has a `weight` | See [Weighting](#weighting). |
| `capabilities_required` | array of strings | No | Defaults to `["standalone_cif"]`. |
| `members` | array | Yes (min 1) | One entry per ensemble member. See [Member](https://www.google.com/search?q=%23member). |
| `metadata` | object | No | Opaque passthrough. |
| `dynamics` | object | No | Opaque passthrough. |

### Terminology

A **member** is a structure-bearing unit included in the ensemble.

A member MAY represent an individual conformer, a trajectory frame, a representative structure, or another structure representation permitted by this contract.

The term **conformational state** MAY be used in domain-specific contexts when a member is understood to represent a conformational state, but `members` is the normative manifest field name.

## `topology_reference`

`topology_reference` identifies the topology against which ensemble members are interpreted.

It MUST contain exactly one of the following.

### `member_id`

A string identifying an existing member:

```yaml
topology_reference:
  member_id: state_001
```

`member_id` MUST match the `id` of an existing entry in `members`.

### `external_reference`

An external topology reference:

```yaml
topology_reference:
  external_reference:
    uri: "..."
    source: "..."
```

`uri` is required.

`source` is optional.

## Member

Each entry in `members` represents one ensemble member.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | string | yes | Unique within the ensemble. |
| `structure` | object | yes | Exactly one of the three structure shapes described below. |
| `weight` | object | no | `{value: number, type?: <weight type>}`. |
| `residue_mapping` | object | no | `{uri: string, format?: string}`. Required if this member's topology differs from `topology_reference`. |
| `thermodynamics` | object | no | Opaque passthrough. |

### Member IDs

Member `id` values MUST be unique within the ensemble.

To ensure cross-platform compatibility and safe URI/filesystem mapping, id strings MUST contain only alphanumeric characters, hyphens, and underscores (matching the regular expression `^[a-zA-Z0-9_-]+$`).

## Structure

A member's `structure` object MUST conform to exactly one of the following three shapes.

### Standalone

A single structure file per member:

```yaml
structure:
  uri: structures/state_001.cif
```

This is the common case.

### Multi-model

One multi-model mmCIF referencing a specific model:

```yaml
structure:
  uri: structures/ensemble.cif
  model_index: 0
```

`model_index` MUST be an integer greater than or equal to `0`.

### Trajectory-backed

A topology plus trajectory referencing a specific frame:

```yaml
structure:
  topology_uri: topology.pdb
  trajectory_uri: trajectory.xtc
  frame_index: 0
  trajectory_format: xtc
```

Required fields:

| Field | Type | Required |
| --- | --- | --- |
| `topology_uri` | string | Yes |
| `trajectory_uri` | string | Yes |
| `frame_index` | integer $\ge 0$ | Yes |
| `trajectory_format` | enum | Yes |

Permitted `trajectory_format` values are strictly lowercase:

```text
xtc
dcd
trr
nc
```

A manifest containing a trajectory-backed member MUST include `"trajectory_backed"` in `capabilities_required`.

## Weighting

`weight_scheme` is required when any member declares a `weight`.

The `weight_scheme.type` MUST be one of:

| Type | Sums to 1.0? | Comparable within ensemble? | Comparable across ensembles? |
| --- | --- | --- | --- |
| `equilibrium_probability` | Yes | Yes | Only if the same generation method and comparable simulation length are used |
| `cluster_fraction` | Yes | Yes | No; depends on clustering algorithm and parameters |
| `experimental_occupancy` | Not necessarily | Qualitatively only | No |
| `uniform` | Yes (all equal) | N/A (means no weighting information beyond uniformity) | N/A |
| `custom` | Not necessarily | Only via `custom_semantics` | No |

### Weighting rules

- `weight_scheme` MUST be present if any member has a `weight`.
- If `weight_scheme.type == "custom"`, `custom_semantics` MUST also be present.
- If a member's `weight.type` is present, it MUST match `weight_scheme.type`.
- A member's `weight.type` MUST NOT contradict the ensemble's `weight_scheme.type`.
- If `weight_scheme.normalized == true`, all member weights MUST sum to `1.0` within a tolerance of `1e-6`.
- For `uniform`, all members are understood to have equal weight. If explicit member weights are provided, they MUST be consistent with uniform weighting.

## Capabilities

`capabilities_required` declares capabilities that a consumer must support to interpret the package.

If any member uses the trajectory-backed structure shape, `capabilities_required` MUST contain:

```text
trajectory_backed
```

If omitted, `capabilities_required` defaults to:

```yaml
capabilities_required:
  - standalone_cif
```

## Residue Mapping

`residue_mapping` identifies a mapping required to interpret a member whose topology differs from `topology_reference`.

Its shape is:

```yaml
residue_mapping:
  uri: mappings/state_001.json
  format: json
```

`uri` is required.

`format` is optional.

A member MUST provide `residue_mapping` when its topology differs from the manifest's `topology_reference`.

## Opaque Fields

Fields explicitly described as **opaque passthrough** are intentionally not interpreted by the PCE core contract.

Their contents MAY be defined by extensions or downstream applications.

The PCE core contract does not assign domain-specific semantics to these fields.

Whether unknown fields are permitted outside explicitly extensible objects is determined by `manifest.schema.json`.

## Semantic Constraints

The following requirements are semantic and may not be fully expressible through JSON Schema alone.

A manifest can therefore satisfy the structural schema while still violating the PCE contract.

A conforming package MUST satisfy all of the following:

- Member `id` values MUST be unique within the ensemble.
- `topology_reference.member_id` MUST reference an existing member.
- `weight_scheme` presence or absence MUST agree with whether any member declares a `weight`.
- Member `weight.type` MUST NOT contradict `weight_scheme.type`.
- If `weight_scheme.normalized == true`, member weights MUST sum to `1.0` within a tolerance of `1e-6`.
- `capabilities_required` MUST list `"trajectory_backed"` if any member uses trajectory-backed structure.
- A member's `residue_mapping` MUST be present when its topology differs from `topology_reference`.
- `content_hash` MUST match the content hash computed for the package according to the PCE content-hash specification.

## Content Integrity

`content_hash` has the form:

```text
algorithm:hexdigest
```

For example:

```text
blake3:ab12...
```

The value identifies a content hash computed over the package according to the applicable PCE content-hash specification.

A syntactically valid hash MUST NOT be treated as evidence that the package contents are correct.

An implementation performing integrity verification MUST recompute the hash from the package contents and compare the result with the manifest value.

The content-hash algorithm, Merkle-tree construction, canonical serialization rules, and resource-byte extraction rules are **not defined by this document**. They are defined by the separate PCE content-hash specification.

The content hash identifies the package's content rather than the particular YAML formatting used to serialize its manifest.

Equivalent YAML and JSON representations of the same PCE data model MUST therefore be capable of representing the same package content without changing its content identity.

## Versioning

`schema_version` uses the form:

```text
X.Y.Z
```

The currently defined version is:

```text
0.1.0
```

Changes to the manifest contract MUST be reflected in the schema version according to the PCE versioning policy.

The precise compatibility guarantees between different schema versions are outside the scope of this document.

## Reproducibility and Scope

The PCE core manifest describes **what an ensemble contains and how its structural contents are interpreted**.

It does not attempt to fully describe **how the ensemble was generated**.

Reproducibility of the package as a data artifact depends on:

- stable identification of the package;
- identification of its structural resources;
- unambiguous interpretation of those resources;
- and verification of `content_hash`.

Workflow provenance, generation parameters, software environments, random seeds, simulation workflows, and derivation histories are outside the scope of the PCE core manifest.

Such information MAY be represented by separate PCE extensions or higher-level workflow/provenance systems.

## Out of Scope

This document defines the data contract, but intentionally excludes the following:

- **Implementation Details:** Tooling for YAML parsing or schema validation.
- **Hashing Mechanics:** BLAKE3 Merkle-tree construction, canonical serialization rules, and trajectory-to-byte extraction methods.
- **Scientific Semantics:** The biochemical interpretation of members as conformational states, the algorithms used to calculate weights, and application-specific logic for opaque objects (`metadata`, `dynamics`, `thermodynamics`).
- **Provenance:** Workflow lineage, generation parameters, and external catalog systems that link an ensemble to its derivation history.

> **Note on Trajectory Hashing:** Because trajectory-backed hashing depends on the separate content-hash specification, trajectory-backed packages MUST NOT be assumed to have interoperable content hashes across implementations until canonical frame extraction is formally defined there.
