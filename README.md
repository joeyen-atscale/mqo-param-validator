# mqo-param-validator

A pure Rust guardrail that rejects an MQO query naming fields the catalog doesn't have — before the engine ever runs it.

`mqo-mcp-server` accepts a structured MQO query but resolves its fields lazily: a `query_multidimensional` call that names a measure which doesn't exist, or a level from the wrong hierarchy, surfaces as a backend error deep in the pipeline instead of a clean rejection at the door. This library closes that gap. Hand it a bound MQO and a catalog snapshot; it tells you, deterministically, which fields don't ground out — and what the caller probably meant instead.

```rust
validate(mqo: &BoundMqoInput, catalog: &CatalogSnapshot) -> Vec<ParamRejection>
```

An empty `Vec` means every field resolved: the query is safe to execute. A non-empty `Vec` means reject it now, one entry per offending field, each carrying ranked suggestions for the likely intent. No LLM, no network, no `unsafe` — the same input always yields the same answer.

## What it checks

Four checks, run over every field in the MQO:

1. **Unmapped field.** Each measure, dimension, and filter must resolve to a catalog `unique_name` after normalization — lowercase, whitespace collapsed, punctuation stripped. An unresolved field is rejected as `Unmapped`, with the nearest catalog names by Jaro-Winkler similarity as suggestions (the typo / lookalike-measure case).

2. **Wrong hierarchy level.** A level that exists somewhere in the catalog but not in the hierarchy implied by its dimension is rejected as `WrongHierarchyLevel`. Suggestions are that hierarchy's actual levels.

3. **Ambiguous date role.** A dimension whose base date dimension backs two or more role-played dates — order date, ship date, due date — with no role qualifier supplied is rejected as `AmbiguousDateRole`, listing every candidate role so the caller can disambiguate.

4. **Cross-fact path** (conservative, structural). When a measure declares a subject area and a referenced dimension is available only in other subject areas, the pair is rejected as `CrossFactPath`. Conformed dimensions — those with an empty `subject_areas` list — are never flagged, so the check produces no false positives on shared dimensions.

The validator is a total function: malformed or partial input yields rejections or an empty result, never a panic.

## Install

A library crate. Add it as a git dependency:

```toml
[dependencies]
mqo-param-validator = { git = "https://github.com/joeyen-atscale/mqo-param-validator" }
```

## Usage

```rust
use mqo_param_validator::{validate, BoundMqoInput, CatalogSnapshot};

let mqo: BoundMqoInput = serde_json::from_str(mqo_json)?;
let catalog: CatalogSnapshot = serde_json::from_str(catalog_json)?;

let rejections = validate(&mqo, &catalog);
if rejections.is_empty() {
    // every field grounded — safe to execute
} else {
    for r in &rejections {
        eprintln!("Rejected [{}] ({:?}): {:?}", r.field, r.class, r.reason);
        for s in &r.suggestions {
            eprintln!("  did you mean: {} (sim={:.3})", s.name, s.similarity);
        }
    }
}
```

## Types

```rust
pub struct CatalogSnapshot {
    pub measures: Vec<CatalogMeasure>,
    pub dimensions: Vec<CatalogDimension>,
    pub hierarchies: Vec<CatalogHierarchy>,
    pub date_roles: Vec<CatalogDateRole>,
}

pub enum FieldClass { Measure, Dimension, HierarchyLevel, DateRole, Filter }

pub enum RejectReason { Unmapped, WrongHierarchyLevel, AmbiguousDateRole, CrossFactPath }

pub struct ParamRejection {
    pub field: String,
    pub class: FieldClass,
    pub reason: RejectReason,
    pub suggestions: Vec<Suggestion>,
}

pub struct Suggestion { pub name: String, pub similarity: f64, pub note: Option<String> }
```

`BoundMqoInput` and the catalog types all derive `serde::Deserialize`, so the JSON you already have from `describe_model` and the bound query maps straight onto them. Every collection field defaults to empty, so partial JSON deserializes without error.

## Acceptance criteria

The validator was built against a fixed acceptance spec (ATSCALE-49213 AC2: "a server-side validator prototype that rejects calls referencing unmapped fields"). Each criterion has a dedicated test file under `tests/`; `cargo test` runs all 29.

| # | Criterion |
|---|-----------|
| AC1 | A fully-mapped MQO returns an empty `Vec`. |
| AC2 | An unmapped measure returns `Unmapped` with ranked nearest-match suggestions. |
| AC3 | A level in the wrong hierarchy returns `WrongHierarchyLevel` with that hierarchy's valid levels. |
| AC4 | A date dimension with ≥2 roles and no qualifier returns `AmbiguousDateRole` with all candidates. |
| AC5 | A measure and dimension from disjoint subject areas return `CrossFactPath`; a conformed dimension does not. |
| AC6 | Malformed or partial input never panics — the validator is total. |
| AC7 | `cargo test` green, `cargo clippy -- -D warnings` clean, zero `unsafe`. |

## Where it fits

Part of the **[mqo-mcp](https://github.com/joeyen-atscale/mqo-mcp)** fleet — the AtScale MQO/MCP engine that serves a semantic model to AI clients. This crate is the pre-execution gate: it turns a class of late backend errors into crisp, typed rejections with repair hints, so a misnamed field is caught and corrected at the boundary rather than failing downstream.

## Dependencies

- `serde` + `serde_json` — catalog and MQO deserialization
- `strsim` — Jaro-Winkler similarity for nearest-match suggestions
