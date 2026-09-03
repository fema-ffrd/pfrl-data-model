# Risk Quantification Methodology (RQM) Data Model

> Re-architecting deterministic building-loss inputs into first-class, **versioned distribution** objects for probabilistic flood risk assessment.

**Version:** 0.1.0 &nbsp;|&nbsp; **License:** [MIT](LICENSE) &nbsp;|&nbsp; **Companion:** `fema-ffrd/inland-consequences` (SPHERE core schemas)

## Overview

Traditional flood-loss estimation treats each building input — flood depth, foundation
type, first-floor height (FFH), and the depth-damage function (DDF) — as a single
deterministic value. The RQM data model instead stores each uncertain input as a typed,
versioned **distribution specification**. A sampling engine draws per-realization values
from those specifications, propagating uncertainty end-to-end into Monte Carlo loss
ensembles with reproducible provenance.

This repository is the schema/data-model deliverable called for in the *Risk Assessment
Maturity Roadmap* (Recommended Immediate Start Activity #1: "Develop data model/schema for
Risk Assessment"). It realizes the roadmap's building-inventory design pillars —
**Reproducible Models, Versioning, Components, Generics,** and **Parametric Distributions** —
as a concrete, validated schema.

### Design goals

1. **Inventory / component coverage** — model structures and their damageable sub-components.
2. **Uncertainty quantification** — replace scalar attributes with distribution objects.
3. **Versioning / provenance** — every distribution, DDF, and run is reproducible and auditable.

## Storage-tier architecture

Storage tiers are inherited from the FFRD data model, each chosen for a distinct workload:

| Tier | Technology | Role |
|---|---|---|
| **Relational** | PostgreSQL | Inventory, distribution specs, DDF/event registries, results metadata, provenance |
| **Lakehouse** | Apache Iceberg | Ensemble-scale realization draws & aggregates (schema evolution, time-travel) |
| **Gridded** | Icechunk / Zarr | N-dimensional gridded hazard fields, referenced by versioned URI |

## Data model

The model is organized into six domains. Tables are defined once in
[data-dictionary/data-dictionary.yaml](data-dictionary/data-dictionary.yaml) (the single
source of truth) and rendered to
[data-dictionary/data-dictionary.md](data-dictionary/data-dictionary.md).

| Domain | Purpose | Key tables |
|---|---|---|
| **A — Inventory** | Structure & component coverage | `buildings`, `building_components`, `generics` |
| **B — Attribute Uncertainty** | Distribution specs for uncertain attributes | `attribute_distributions`, `foundation_pmf` |
| **C — Hazard Linkage** | Events and probabilistic flood depth | `events`, `hazard_links` |
| **D — Depth-Damage Functions** | Versioned, probabilistic DDFs | `ddf_library`, `ddf_uncertainty` |
| **E — Realization & Loss Results** | Ensemble-scale draws & summaries | `loss_realizations`, `mv_loss_summary` |
| **F — Provenance & Versioning** | Reproducibility & lineage | `run_catalog`, `manifests`, `run_logs`, `versioning` |

### How the pieces fit together

- **`buildings`** anchors the inventory. Immutable/base attributes live here; *uncertain*
  attributes do not — they live in **`attribute_distributions`** and are resolved per draw.
- **`building_components`** decomposes a structure into modular sub-assemblies (finish,
  foundation, structure, contents, inventory), typed by the extensible **`generics`**
  vocabulary so non-building inventories can be added without a schema break.
- **`foundation_pmf`** captures the "shuffled" foundation type as an explicit probability
  mass function plus a reproducible shuffle policy.
- **`events`** is the canonical registry of hazard scenarios; its `event_id` is referenced
  by `run_catalog`, `hazard_links`, and `loss_realizations`.
- **`hazard_links`** ties a structure to a versioned gridded depth surface (Icechunk/Zarr)
  and its depth-in-structure uncertainty, rather than copying a scalar depth into the row.
- **`ddf_library` / `ddf_uncertainty`** make the DDF itself a distribution: a realization
  draws a DDF percentile instead of a single mean curve.
- **`loss_realizations`** (Iceberg) holds one row per building × event/AEP × Monte Carlo
  draw, each reproducible from its `seed` plus version pointers; **`mv_loss_summary`**
  pre-computes central tendency and upper prediction limits.
- **`versioning`** is the central lineage registry applied to every distribution, PMF, DDF,
  event, and inventory snapshot.

### Entity-relationship diagram

The ERD is maintained in [diagrams/rqm-erd.mmd](diagrams/rqm-erd.mmd) (Mermaid). It is an
abridged view — it shows keys and relationships, not every column. Refer to the data
dictionary for the authoritative column list. GitHub renders `.mmd` files automatically;
locally you can preview it with any Mermaid-capable viewer.

## Repository structure

```
rqm-data-model/
├── data-dictionary/
│   ├── data-dictionary.yaml   # single source of truth (tables, columns, enums, FKs)
│   ├── data-dictionary.md     # generated — do not edit by hand
│   ├── preview_dict.py        # validates FKs and renders the Markdown
│   └── requirements.txt       # generator dependencies (PyYAML)
├── diagrams/
│   └── rqm-erd.mmd            # Mermaid entity-relationship diagram
├── docs/
│   └── Risk_Assessment_Maturity_Roadmap - DRAFT.pdf   # source roadmap
└── LICENSE
```

## Regenerating the data dictionary

The YAML is authoritative; the Markdown is generated. After editing the YAML, regenerate and
validate in one step:

```powershell
pip install -r data-dictionary/requirements.txt
python data-dictionary/preview_dict.py
```

The generator validates that every PostgreSQL table has a primary key and that every foreign
key points to an existing `table.column`, then writes `data-dictionary.md`. It exits non-zero
on any validation failure, so it is safe to run in CI. Expected output:

```
OK: validated 15 tables, 132 columns. Wrote data-dictionary.md.
```

## Roadmap alignment & maturity status

The model maps to the six workstreams of the *Risk Assessment Maturity Roadmap*. Coverage of
each roadmap concept by the current schema:

| Roadmap concept | Status | Realized by |
|---|---|---|
| Reproducible models | ✅ Covered | `run_catalog`, `manifests`, `run_logs`, `loss_realizations.seed` |
| Versioning | ✅ Covered | `versioning` |
| Components | ✅ Covered | `building_components` |
| Generics (extensible inventory) | ✅ Covered | `generics` |
| Uncertainty / parametric distributions | ✅ Covered | `attribute_distributions`, `foundation_pmf`, `ddf_uncertainty`, `hazard_links` |
| DDF library & assignment | ✅ Covered | `ddf_library` |
| Hazard / flood-depth linkage | ✅ Covered | `hazard_links`, `events` (+ Icechunk) |
| Monte Carlo convergence criteria | 🟡 Partial | `run_catalog.n_realizations`, `mv_loss_summary.loss_cv` (no explicit convergence record) |
| Coastal (wave / SWL / compound) | 🟡 Partial | `peril_type`, `hazard_links.velocity_grid_uri` / `duration_grid_uri` |
| Global sensitivity analysis | 🟡 Partial | `run_type = sensitivity` only |
| Decision uncertainty / BCA | ⚪ Downstream | Out of scope — consumed by BCA tooling |

## License

Released under the [MIT License](LICENSE).
