# Protein Ensemble — Data Contract Reference (v1.0.0)

Plain-English companion to `manifest.schema.json`. This is the shape only
— no implementation detail. See the reference implementation (`pce`
Python library) for parsing/validation/hashing code.

A PCE package = one `manifest.yaml` + the structure files it references,
in a directory tree. `manifest.yaml`'s top-level fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string, `"X.Y.Z"` | yes | Currently `"1.0.0"`. |
| `id` | string | yes | Ensemble ID. |
| `content_hash` | string, `"algo:hexdigest"` | yes | e.g. `"blake3:ab12..."`. Merkle hash over all members; must be recomputed and verified on load, not trusted from the file. |
| `parent_ensemble` | object or `null` | yes (always present) | `null` for a root ensemble. Never omit this key even when there's no parent. |
| `topology_reference` | object | yes | Exactly one of `member_id` or `external_reference`. |
| `weight_scheme` | object | only if any member has a `weight` | See Weighting below. |
| `capabilities_required` | array of strings | no (defaults to `["standalone_cif"]`) | Must include `"trajectory_backed"` if any member uses trajectory-backed structure. |
| `members` | array | yes, min 1 | See Member shape below. |
| `metadata` | object | no | Opaque passthrough. |
| `dynamics` | object | no | Opaque passthrough. |

## `parent_ensemble` (object)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Parent ensemble's ID. |
| `content_hash` | string | Parent's content hash, same format as top-level. |
| `relationship` | string | Free text describing the derivation (e.g. `"adaptive_sampling_round_2"`). |

## `topology_reference` (object)

Exactly one of:

- `member_id` (string) — must match an existing member's `id`.
- `external_reference` (object): `{uri: string, source?: string}`.

## Member (object, one entry per `members[]`)

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | string | yes | Unique within the ensemble. |
| `structure` | object | yes | One of three shapes — see below. |
| `weight` | object | no | `{value: number, type?: <weight type>}`. |
| `residue_mapping` | object | no | `{uri: string, format?: string}`, required if this member's topology differs from `topology_reference`. |
| `provenance` | object | no | Opaque passthrough. |
| `thermodynamics` | object | no | Opaque passthrough. |

### `structure` — three mutually exclusive shapes

**Standalone** (single file per member, the common case):

```
{uri: string}
```

**Multi-model** (one multi-model mmCIF, referencing one model):

```
{uri: string, model_index: integer >= 0}
```

**Trajectory-backed** (topology + trajectory, referencing one frame):

```
{
  topology_uri: string,
  trajectory_uri: string,
  frame_index: integer >= 0,
  trajectory_format: "xtc" | "dcd" | "trr" | "nc"
}
```

Requires `"trajectory_backed"` in the manifest's top-level
`capabilities_required`.

## Weighting

`weight_scheme.type` is one of:

| Type | Sums to 1.0? | Comparable within ensemble? | Comparable across ensembles? |
| --- | --- | --- | --- |
| `equilibrium_probability` | yes | yes | only if same generation method + comparable sim length |
| `cluster_fraction` | yes | yes | no (depends on clustering algo/params) |
| `experimental_occupancy` | not necessarily | qualitatively only | no |
| `uniform` | yes (all equal) | n/a — means "no weighting info" | n/a |
| `custom` | not necessarily | only via `custom_semantics` | no |

- `weight_scheme` is required the moment *any* member declares a `weight`.
- If `weight_scheme.type == "custom"`, `custom_semantics` (object) is also required.
- A member's `weight.type`, if present, must match the ensemble's `weight_scheme.type` — they can't disagree.
- If `weight_scheme.normalized == true`, all member weights must sum to 1.0 (±1e-9).

## Rules a schema validator alone won't catch

These are semantic, not structural — a document can pass JSON Schema and
still violate one of these:

- Member `id`s must be unique within the ensemble.
- `topology_reference.member_id` must reference a member that actually exists.
- `weight_scheme` presence/absence must match whether any member has a `weight`.
- Member `weight.type` must not contradict `weight_scheme.type`.
- Normalized weights must actually sum to 1.0.
- `capabilities_required` must list `"trajectory_backed"` if any member uses that structure shape.
- `content_hash` must match a recomputed hash over the actual on-disk structure files — a syntactically valid hash string doesn't mean it's the *correct* one.

## What this doesn't cover

- The actual `content_hash` algorithm (BLAKE3 Merkle tree construction) — implementation detail, not contract.
- Canonical YAML serialization rules used to compute that hash — same.
- How to extract "frame bytes" from a trajectory format for hashing — unresolved even in the reference implementation; treat trajectory-backed hashing as an open question, not a settled part of the contract.
