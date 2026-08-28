# Protein Ensemble (PE) Manifest Specification

![Schema Version](https://img.shields.io/badge/schema-v1.0.0-blue.svg)

This repository defines the core **data contract and semantic requirements** for the Protein Ensemble (PE) manifest.

A PCE package consists of a `manifest.yaml` file alongside the structural resources it references (such as `.cif`, `.pdb`, or `.xtc` files), organized within a standard directory tree. This specification standardizes what an ensemble contains and how its structural contents are interpreted.

## Repository Contents

- `manifest.schema.json`: The normative JSON Schema (Draft-07) defining the strict structural contract of the `manifest.yaml`.
- `pce-manifest-contract.md`: The plain-English semantic specification. It details rules that cannot be fully expressed in JSON Schema (such as cross-field validation, weight tolerances, and URI security constraints).

## Scope

The PE core manifest describes **what an ensemble contains and how its structural contents are interpreted**.

It **does not** define:

- Implementation tooling (e.g., YAML parsers, validation code).
- Hashing mechanics (e.g., BLAKE3 Merkle-tree construction or trajectory-to-byte extraction).
- Scientific semantics (e.g., calculating weights, workflow provenance, or molecular interpretations).

For details on out-of-scope elements, refer to the [Semantic Specification](pce-manifest-contract.md).

## 📦 Package Structure

A minimal PCE package has the following form:

```text
ensemble_001/
├── manifest.yaml
└── structures/
    ├── state_001.cif
    ├── state_002.cif
    └── state_003.cif
