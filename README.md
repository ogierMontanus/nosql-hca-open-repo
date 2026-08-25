# HCA Open Repository — database layer (draft)

This repo is the continuation of the PostgreSQL data-layer drafting work
originally started in
[`hca-open-repo`](https://github.com/ogiermontanus/hca-open-repo), the
main HCA Open Repository (formerly "H.C. Andersen Dagbogsregister")
project. That repo's data currently lives as CSV/Excel/TSV files; this
repo is where the design and eventual implementation of the database
that replaces (or sits underneath) that CSV layer is drafted and built.


## Documents

| File | Purpose |
|---|---|
| [`docs/data-model/postgres-schema-design.md`](docs/data-model/postgres-schema-design.md) | Proposed normalized PostgreSQL schema for the Person / Work / Place registers — analysis of the source CSVs, conceptual model, full `CREATE TABLE` statements, CSV→Postgres column mapping, data-quality issues, editorial workflow, and open questions |
| [`docs/data-model/postgres-schema-design-addendum-parsed-works.md`](docs/data-model/postgres-schema-design-addendum-parsed-works.md) | Companion addendum: what `data/parsed/*.tsv` (the richest work-entity data in the source repo) change in the schema above |

## Relationship to `hca-open-repo`

- **Source of truth for data, for now:** the register CSVs
  (`data/normalized/`, `data/curated/`, `data/parsed/`) and the live
  CSV/Excel-driven mockup stay in `hca-open-repo`. This repo does not
  duplicate that data.
- **Signpost:** `hca-open-repo`'s
  `docs/data-model/postgresql-migration.md` points here for anyone
  looking for the PostgreSQL draft from that side.
- **Star schema / analytical model:** the dimension/fact (Power Query /
  Power Pivot) model that these documents complement remains documented
  in `hca-open-repo`'s `docs/data-model/star-schema.md` — this repo's
  schema is the authoritative/editable (OLTP) layer underneath it, not a
  replacement for it.

## Status

Proposal stage. Not yet built or reviewed with the project team. See the
open questions section (§H) of `postgres-schema-design.md` for the
decisions still needed before implementation starts.
