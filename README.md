# HCA Open Repository — database layer (draft)

This repo is the continuation of the PostgreSQL data-layer drafting work
originally started in
[`hca-open-repo`](https://github.com/ogiermontanus/hca-open-repo), the
main HCA Open Repository (formerly "H.C. Andersen Dagbogsregister")
project. That repo's data currently lives as CSV/Excel/TSV files; this
repo is where the design and eventual implementation of the database
that replaces (or sits underneath) that CSV layer is drafted and built.

Despite the repo name (`nosql-hca-open-repo`), the drafted schema is a
normalized **PostgreSQL** design, not a NoSQL/document-store one — see
[`docs/data-model/postgres-schema-design.md`](docs/data-model/postgres-schema-design.md)
for the current status and rationale.

## Documents

| File | Purpose |
|---|---|
| [`docs/data-model/postgres-schema-design.md`](docs/data-model/postgres-schema-design.md) | Proposed normalized PostgreSQL schema for the Person / Work / Place registers — analysis of the source CSVs, conceptual model, full `CREATE TABLE` statements, CSV→Postgres column mapping, data-quality issues, editorial workflow, and open questions |
| [`docs/data-model/postgres-schema-design-addendum-parsed-works.md`](docs/data-model/postgres-schema-design-addendum-parsed-works.md) | Companion addendum: what `data/parsed/*.tsv` (the richest work-entity data in the source repo) change in the schema above |
| [`docs/data-model/postgres-schema-design-addendum-hca-db-persons.md`](docs/data-model/postgres-schema-design-addendum-hca-db-persons.md) | Companion addendum: the **person crosswalk** — mapping `hca_db`'s 2,018-row `brevperson` correspondence register onto `hca-open-repo`'s 10,228 person entities, with the `person_external_map` mapping table, match tiers, measured hold-out error rates, and the letter layer that hangs off it |
| [`docs/data-model/postgres-schema-design-addendum-hca-db-works.md`](docs/data-model/postgres-schema-design-addendum-hca-db-works.md) | Companion addendum: the **works crosswalk** — `hca_db`'s Index A (BFN, 1,274 publications) and Index B (Værkfortegnelsen, 1,508 works) against `hca-open-repo`'s VÆRK-REGISTER, mapped through the project's simplified WEMI rule; includes the BFN attribution correction and the WEMI-level mismatch that caps title matching |
| [`docs/data-model/minimal-core-and-jsonb-payload.md`](docs/data-model/minimal-core-and-jsonb-payload.md) | The team's **minimal-obligatory-core + structured-JSON** recommendation worked out: the boundary rule for what may go in the payload and what must stay relational, measured fill rates behind "obligatory", a validated `entity_core` table, and a full worked person entry (Edvard Collin — 767 diary occurrences, 742 letters) |

## Relationship to `hca-open-repo`

- **Source of truth for data, for now:** the register CSVs
  (`data/normalized/`, `data/curated/`, `data/parsed/`) and the live
  CSV/Excel-driven mockup stay in `hca-open-repo`. This repo does not
  duplicate that data.
- **Signpost:** `hca-open-repo`'s
  `docs/data-model/postgresql-migration.md` points here for anyone
  looking for the PostgreSQL draft from that side.
- **A third repo, `hca_db_export`:** a read-only snapshot of the
  SDU `hca_db` MySQL database (dumped 2023-06-29) plus the crosswalk
  specification in its `docs/`. It is an **external authority mapped
  against** `hca-open-repo`, never a second master — see the person
  addendum's governing principle. This repo holds the schema that
  expresses that mapping.

- **Star schema / analytical model:** the dimension/fact (Power Query /
  Power Pivot) model that these documents complement remains documented
  in `hca-open-repo`'s `docs/data-model/star-schema.md` — this repo's
  schema is the authoritative/editable (OLTP) layer underneath it, not a
  replacement for it.

## Status

Proposal stage. Not yet built or reviewed with the project team. See the
open questions section (§H) of `postgres-schema-design.md`, and §10 of the
person addendum, for the decisions still needed before implementation
starts.

The DDL across all three documents has been executed end to end against
PostgreSQL 16 and the person-crosswalk constraints behaviour-tested — see
the person addendum §7b. That makes it a *validated* proposal, not an
implemented system: no data has been loaded.
