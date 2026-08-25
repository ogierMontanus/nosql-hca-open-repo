# PostgreSQL schema design — Person / Work / Place registers

Status: proposal, not yet built or reviewed with the project team.

> **Provenance:** moved here from
> [`hca-open-repo`](https://github.com/ogiermontanus/hca-open-repo)'s
> `docs/data-model/postgres-schema-design.md` on 2026-08-25 — this repo is
> now the home for the continued drafting of the PostgreSQL data layer.
> The source CSVs this document analyses (`data/normalized/`,
> `data/curated/`, `data/parsed/`) still live in `hca-open-repo`; see that
> repo's `docs/data-model/postgresql-migration.md` for the signpost back.

This document analyses the repo's actual register CSVs and proposes a
normalized PostgreSQL schema for them. It complements, rather than
replaces, [`star-schema.md`](star-schema.md): that document frames the
**analytical** (dimension/fact, Power Pivot → warehouse) shape of the same
data; this document proposes the **authoritative, editable, OLTP-style**
shape underneath it — the database a scholarly editor writes to, which the
star schema (or a materialized view/ETL job over it) can then be built
from. Where the two disagree, this document defers to `star-schema.md`'s
already-agreed framing (Registry parent + Person/Work/Place sub-types,
JSONB for heterogeneous attributes, "thin layer" principle) and explains
the relationship explicitly rather than silently diverging.

All numbers below were counted directly from the CSVs in this repo
(`data/normalized/`, `data/curated/`) on 2026-08-19, not estimated.

> **Three companion addenda extend this document; each corrects or adds
> to what is below.**
>
> - [`…-addendum-parsed-works.md`](postgres-schema-design-addendum-parsed-works.md)
>   — `data/parsed/*.tsv`, the richest work-entity data in the source repo.
> - [`…-addendum-hca-db-persons.md`](postgres-schema-design-addendum-hca-db-persons.md)
>   — the **person crosswalk** against the `hca_db` MySQL dump in
>   `hca_db_export`: a second, external person register (`brevperson`,
>   2,018 rows) mapped onto this document's `person` table through a
>   `person_external_map` table, plus the correspondence layer. It also
>   partially answers §H.8 below: the dump's `sted` table is **empty**
>   (1 row), while its `almanak` table has 5,163.
> - [`…-addendum-hca-db-works.md`](postgres-schema-design-addendum-hca-db-works.md)
>   — the **works crosswalk** against the same dump: Index A (BFN) and
>   Index B (Værkfortegnelsen) mapped onto this document's `work` table
>   through `hca-open-repo`'s simplified WEMI rule. It scopes the
>   crosswalk to the register's 768-row H. C. ANDERSEN wing (the other
>   2,940 works have no counterpart in that source at all) and shows why
>   the register's Work/Manifestation mixing, not title noise, is what
>   caps matching.

---

## 1. Analysis of the source data

### 1.1 `data/normalized/entities.csv` — the master register (16,444 rows)

One denormalized table holding **all three registers at once**,
distinguished by `entity_type`:

| entity_type | category_h1 | rows |
|---|---|---|
| `person` | `PERSON-REGISTER` | 10,228 |
| `work` | `VÆRK-REGISTER` | 3,708 |
| `place` | `STED-REGISTER` | 2,508 |

Columns: `entity_id, entity_type, category_h1, genre_h2, form_h3,
subform_h4, label, description, see, see_also, year_derived, date_derived,
person_derived`.

**Evidence that this is a type-conditional wide table, not a uniform
one** — fill rate by `entity_type`:

| column | person | work | place |
|---|---|---|---|
| `genre_h2` / `form_h3` | 0% | 100% | 0% |
| `subform_h4` | 0% | 17% | 0% |
| `description` | 92% | 0% | 0% |
| `see` / `see_also` | 0% | 3% / 2% | 0% |
| `year_derived` / `date_derived` | 0% | 12% / 6% | 0% |
| `person_derived` | 0% | 37% | 0% |

In other words: `genre_h2/form_h3/subform_h4` (the H2/H3/H4 category
hierarchy), `see/see_also`, `year_derived/date_derived`, and
`person_derived` (the work's inferred author) are **work-only** columns
that happen to sit in the same table as person and place rows, which
never populate them. `description` is (almost) **person-only**. Places
carry nothing beyond `entity_id`/`label` in this file — every other
place attribute (coordinates, country, GeoNames/Wikipedia links) lives in
a separate curated overlay (§1.5).

**`entity_id`** (`Reg0000000`–`Reg00xxxxx`, 7 digits, zero-padded) is a
single ID space shared across all three entity types — confirmed unique
across the full 16,444-row file (no collisions between a person, a work
and a place sharing an ID). This is the natural primary key to preserve
as an external/legacy identifier; see §3's `entity.legacy_reg_id`.

**Embedded relationships found in free-text columns, not modeled as
columns:**

- `person_derived` (a work's author) is free text, not a foreign key —
  and is documented elsewhere in the repo (`CLAUDE.md`,
  `billedkunst-artist-extraction.md`) as sometimes containing **more than
  one name on separate lines** ("158 works across every wing carry a
  `person_derived` with an embedded multiline value"). This column is an
  *unresolved* author reference, not a normalized link — see the
  `work_author` junction table in §3, which keeps the raw text alongside
  a nullable resolved `person_id`.
- `see` / `see_also` are themselves sometimes empty while the **label
  itself embeds a redirect**: this session's own earlier work on
  `persons.html` (documented in
  [`person-bio-search-links.md`](person-bio-search-links.md)) found ~630
  of 10,228 person labels carry a leading `"– Se også: X"` or trailing
  `", se: X"` cross-reference *inside the label string*, not in a
  structured column — e.g. `"– Se også: Hansen, Magdalene."`. The same
  convention is documented for works in
  `scripts/build_mockup/build_works_extra.py`'s `SEE_TAIL_RE`. This is
  clear evidence that the register's own editorial convention for
  cross-references predates, and is inconsistent with, the `see`/
  `see_also` columns — both forms need to be captured (see
  `entity_cross_reference` in §3).

### 1.2 `data/normalized/references.csv` — occurrence data (69,405 rows)

Columns: `page_id, entity_id, entity_label, vol, page, seq`.

This is the **index proper**: one row per (diary page × register entry)
occurrence — exactly the "occurrence in a text" the brief distinguishes
from entity data. It is *not* an entity table.

**Redundancy found:** `entity_label` duplicates `entities.label` for the
referenced `entity_id` (a denormalized cache, presumably for
spreadsheet-era convenience) and `vol`/`page` duplicate what `page_id`
already encodes (`page_id` values like `Pag010067` map 1:1 to a
`vol`+`page` pair via `kb_diary_links.csv`, confirmed for 4,412 of 4,413
rows there — see §1.4). Neither should be carried into normalized tables
as stored columns; both are one join away.

### 1.3 `data/normalized/diary.csv` — diary text (2,177 rows)

Columns: `vol, page, date, month, year, heading, text`.

This is **line/paragraph-level transcribed text**, not one row per page —
multiple rows share the same `vol`+`page` (a page's text arrives as
several dated/undated fragments, each carrying its own `text` chunk with
inline line-number markers like `"001-01"`). Only volumes VI–VII
currently carry structured `date`/`month`/`year` per row; other volumes
would need the same bibliographic fallback (`vol`+`page`) the mockup
already uses. This file is the raw transcription layer behind the
generated `mockup/data/diary-index.js`/`diary-pages/` used by the current
site — the schema in §3 keeps diary text at this same line/section grain
(`diary_line`) rather than collapsing it to one page = one row, since the
source data itself is not at page grain.

### 1.4 `data/normalized/kb_diary_links.csv` — external facsimile links (4,413 rows)

Columns: `pag_id, vol, page, kb_url, source`. A clean 1:1 lookup from a
diary page to its Det Kgl. Bibliotek facsimile URL — this is a **source
reference at the diary-page level**, structurally simple, no ambiguity
found beyond the one known discrepancy already documented in
`docs/external-links.md` (Bind I side 13 pointing at side 32's file in
the source workbook).

### 1.5 Curated overlay CSVs — authority identifiers, not source data

`data/curated/works_wikidata.csv` (35 rows: `rid, wd, image_filename,
attribution_note, verified_via, notes`) and
`data/normalized/sv14_places_reconciled.csv` (63 rows: `entity_id,
label_da, matched_name_en, match_method, lat, lon, sv14_xml_id,
geonames_url, wiki_url`) plus its sibling
`sv14_places_ambiguous.csv` (1 row today: `entity_id, label_da,
candidates` — a place with **more than one** candidate geocode, not yet
resolved) are this project's own hand- and pipeline-curated authority
links, added *after* the original register was digitized — **not** part
of the original source. Crucially, `works_wikidata.csv` already carries
its own **provenance columns** (`verified_via`, `notes`) — direct,
in-repo evidence that per-identifier provenance tracking (§5) is not a
speculative requirement, it is how this project already works, just not
yet in a database.

**Authority-identifier coverage is sparse and per-type, not uniform:**
35 of 3,708 works (~0.9%) have a Wikidata id; 63 of 2,508 places (~2.5%)
have coordinates/GeoNames/Wikipedia; 0 of 10,228 persons have any
authority id in the current CSVs (`persons_wikidata.csv` referenced
elsewhere in the repo's code does not yet exist on disk — the loader that
reads it degrades to empty). This confirms authority linking must be
modeled as an **optional, sparse, per-entity** relationship (§3's
`authority_identifier`), not as columns on `person`/`work`/`place`.

### 1.6 Derived/inferred attribute CSVs — NOT source data, this project's own NLP output

Several CSVs are outputs of this project's own heuristic parsers over
`entities.description`, each carrying explicit confidence/method/
explanation columns already — i.e. the source data model already
distinguishes "what the register says" from "what we inferred from what
the register says":

| CSV | Rows | Grain | Confidence/provenance columns present |
|---|---|---|---|
| `person_gender.csv` | 10,228 (1 per person) | person | `confidence, konflikt, antal_indikatorer, indikatorer, forklaring` |
| `person_gender_review.csv` | 300 (subset) | person | + `menneskelig_vurdering, kommentar` — a genuine **human review queue**, not a separate fact |
| `person_role.csv` | 10,228 rows, 5,564 non-empty | person | `roller` is **semicolon-separated multi-value** (e.g. `"Akademiker/Lærd;Kunstner/Billedkunst"`), plus `kilde_vaerk, kilde_beskrivelse_termer` recording *why* |
| `person_ethnic_descriptors.csv` | 2,703 rows over 2,517 distinct persons (165 persons have 2 rows) | **mention**, not person | `match_type, position_index, position_type, referent_hint` — see below |
| `work_languages.csv` | 1,084 of 3,708 works | work | `method, confidence` |
| `ner_page_grounding.csv` / `_review.csv` | 11,581 / 9,569 rows | **character-offset mention** within a diary page's text | `confidence, method`; includes negative results (`method='no_match'`, `confidence=0.0`, empty `mention_text`) |

The `referent_hint` column in `person_ethnic_descriptors.csv` is
important and easy to miss: a nationality adjective matched in a
person's `description` does not always describe *that* person — the
project's own `nation.html` documentation gives the example "g. m.
*svensk* Premierløjtnant" (describing a spouse, not the register
subject). This is direct evidence that nationality-mention data must stay
a **mention table** (`person_nationality_mention` in §3), never collapsed
into a single `person.nationality` column, or the schema would silently
assert nationality claims the source data itself flags as uncertain.

`ner_page_grounding.csv`'s `ref_page_id` values are a 100%-overlapping
subset of `references.csv`'s `page_id` space (verified: all 10,628
distinct `ref_page_id` values in the grounding file also appear as
`page_id` in `references.csv`). This strongly suggests the grounding file
is a **finer-grained companion** to `references.csv` — attempting to
locate the exact text span for occurrences `references.csv` already
lists — but the CSVs alone do not prove which table is upstream of the
other, or whether `references.csv` is meant to be regenerated from
grounding review. Flagged as Open Question (§H).

### 1.7 Controlled-vocabulary CSVs

`data/curated/nation_umbrellas_da.csv` (18 rows: `umbrella_key,
umbrella_label, place_label_da, members, notes`) is a genuine
many-to-many crosswalk, not a tree: `members` is a semicolon-separated
list of ethnic/national adjective keys, and at least one key
(`holstensk`) deliberately belongs to **two** umbrellas (`dansk` and
`tysk`) at once, per the CSV's own documented rationale (the duchies were
contested territory, 1848–51/1864). This directly justifies a
many-to-many junction table (§3's `nation_umbrella_member`), not a
single `parent_umbrella` foreign key on the adjective.

---

## 2. Conceptual model

### 2.1 Supertype/subtype for Person/Work/Place

The CSV evidence (§1.1) — one shared `entity_id` space, one shared
`label` column, type-conditional columns for everything else — matches
exactly the "Registry parent + Work/Person/Place sub-type" pattern
`star-schema.md` already establishes for the Excel/Power Pivot model
(`PKRegistryTitelID` shared by parent and sub-type rows). This schema
adopts the same pattern in Postgres: an `entity` supertype table holding
what's common (id, type, label, editorial/audit metadata), and
`person`/`work`/`place` subtype tables in a 1:1 relationship with it,
holding only the columns evidenced for that type. This is not a new
design choice invented for this document — it is the same shape the
project already agreed on for the analytical layer, applied to the
authoritative layer underneath it.

### 2.2 Tables justified by the CSVs

| Table | Justified by |
|---|---|
| `entity`, `person`, `work`, `place` | §1.1 (shared ID space, type-conditional columns) |
| `diary_page`, `diary_line` | §1.3 (`diary.csv`'s line grain), §1.4 (`kb_diary_links.csv`) |
| `entity_occurrence` | §1.2 (`references.csv`) |
| `entity_mention` | §1.6 (`ner_page_grounding.csv`'s character-offset grain, distinct from `entity_occurrence`'s page grain) |
| `entity_cross_reference` | §1.1 (`see`/`see_also` columns **and** the label-embedded redirect convention) |
| `work_author` | §1.1 (`person_derived`, including its documented multiline/multi-author cases) |
| `authority_identifier` | §1.5 (`works_wikidata.csv`, `sv14_places_reconciled.csv`, sparse/per-type coverage, existing `verified_via` provenance) |
| `person_gender_assessment` | §1.6 (`person_gender.csv` + `_review.csv`, merged — see §4) |
| `person_role` (lookup) + `person_role_assignment` (junction) | §1.6 (`person_role.csv`'s semicolon-separated `roller`) |
| `ethnic_adjective` (lookup) + `person_nationality_mention` | §1.6 (`person_ethnic_descriptors.csv`'s mention grain and `referent_hint`) |
| `nation_umbrella` + `nation_umbrella_member` | §1.7 (`nation_umbrellas_da.csv`'s M:M `members` with documented dual membership) |
| `work_language_assessment` | §1.6 (`work_languages.csv`) |
| `import_batch` | §5 — every derived-attribute CSV already timestamps/versions itself informally (filenames, `_review` suffixes); formalizing this as a table is the direct database equivalent |

### 2.3 Tables considered and **rejected** for lack of evidence

- **`work_place` / `person_place` relationship tables.** No column in
  any inspected CSV links a work or person to a place directly.
  `star-schema.md` names the intended source for this edge — "Sørens 2.
  register" — as **not yet delivered** (its own Open Question #1). The
  mockup's "Hyppigste steder"/"Hyppigst samtidige steder" features are
  *computed* by co-occurrence on a shared `page_id` in
  `entity_occurrence`, not sourced from a native relationship. In
  Postgres this stays a query (or materialized view) over
  `entity_occurrence`, not a stored table — building one now would be
  exactly the "table created because it's theoretically useful" the
  brief warns against. Revisit once the Sørens register actually
  arrives.
- **A single flat `nationality` column on `person`.** Rejected per
  §1.6's `referent_hint` finding — the source data itself marks some
  matches as not describing the register subject.
- **Separate tables for `person_gender.csv` and
  `person_gender_review.csv`.** The `_review.csv` file is a same-grain
  subset (300 of the same 10,228 persons) needing human attention, not a
  different entity — modeled as extra nullable columns on one table
  (§4), not a second table.

---

## 3. PostgreSQL schema

Design choices explained inline; full rationale for ID types, indexing,
and partitioning is in §8.

```sql
-- ============================================================
-- Extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- fuzzy/ILIKE search on labels

-- ============================================================
-- Controlled vocabularies
-- ============================================================
CREATE TYPE entity_type AS ENUM ('person', 'work', 'place');
CREATE TYPE editorial_status AS ENUM ('draft', 'published', 'needs_review', 'redirect');
CREATE TYPE cross_reference_type AS ENUM ('see', 'see_also');
CREATE TYPE gender_value AS ENUM ('mandlig', 'kvindelig', 'endnu_ubestemt');
CREATE TYPE referent_hint AS ENUM ('subject', 'spouse', 'relative', 'institution', 'other');
CREATE TYPE match_position AS ENUM ('leading', 'embedded', 'trailing');
CREATE TYPE reconciliation_status AS ENUM ('candidate', 'confirmed', 'rejected');

-- ============================================================
-- Import batches — provenance for every row's origin (§5)
-- ============================================================
CREATE TABLE import_batch (
  batch_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source_file    TEXT NOT NULL,              -- e.g. 'data/normalized/entities.csv'
  source_sha256  TEXT,                       -- checksum of the CSV at import time
  imported_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  imported_by    TEXT NOT NULL,              -- person or pipeline that ran the import
  notes          TEXT
);

-- ============================================================
-- Entity supertype (§2.1)
-- ============================================================
CREATE TABLE entity (
  entity_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  legacy_reg_id   TEXT NOT NULL UNIQUE,        -- preserves 'Reg0030000' etc.
  entity_type     entity_type NOT NULL,
  label           TEXT NOT NULL,
  editorial_status editorial_status NOT NULL DEFAULT 'published',
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX entity_label_trgm_idx ON entity USING gin (label gin_trgm_ops);
CREATE INDEX entity_type_idx ON entity (entity_type);

-- Cross-references: both the structured see/see_also columns AND the
-- label-embedded "– Se også:"/", se:" convention resolve into this one
-- table (§1.1). to_entity_id is nullable because the target does not
-- always resolve to a real register row (a name-only pointer).
CREATE TABLE entity_cross_reference (
  cross_reference_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  from_entity_id  BIGINT NOT NULL REFERENCES entity(entity_id),
  to_entity_id    BIGINT REFERENCES entity(entity_id),
  to_text_raw     TEXT NOT NULL,      -- the unresolved label text, always kept
  reference_type  cross_reference_type NOT NULL,
  source_field    TEXT NOT NULL,      -- 'see' | 'see_also' | 'label_embedded'
  UNIQUE (from_entity_id, to_text_raw, reference_type)
);
CREATE INDEX xref_from_idx ON entity_cross_reference (from_entity_id);
CREATE INDEX xref_to_idx ON entity_cross_reference (to_entity_id) WHERE to_entity_id IS NOT NULL;

-- ============================================================
-- Person subtype
-- ============================================================
CREATE TABLE person (
  entity_id     BIGINT PRIMARY KEY REFERENCES entity(entity_id),
  description   TEXT,       -- the free-text register note (entities.description)
  born_year     SMALLINT,   -- see §5/§D — not present as a structured column
  died_year     SMALLINT    -- today; parsed from the label's "(1825–1912)" if needed
);

-- ============================================================
-- Work subtype
-- ============================================================
-- genre_h2/form_h3/subform_h4 are a closed, low-cardinality hierarchy
-- (100%/100%/17% fill on works only) — normalized into a lookup rather
-- than kept as free text, so a no-code layer can render it as a dropdown
-- and so mis-typed category values become impossible.
CREATE TABLE work_category (
  category_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  h2            TEXT NOT NULL,
  h3            TEXT,
  h4            TEXT,
  UNIQUE (h2, h3, h4)
);

CREATE TABLE work (
  entity_id     BIGINT PRIMARY KEY REFERENCES entity(entity_id),
  category_id   BIGINT REFERENCES work_category(category_id)
);

-- year_derived is sometimes multi-valued in the source ("1847, 1872" —
-- publication + reprint) so it is a child table, not a column.
CREATE TABLE work_date (
  work_date_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id     BIGINT NOT NULL REFERENCES work(entity_id),
  year          SMALLINT,
  date_text     TEXT,        -- date_derived, kept as text: precision varies
  date_role     TEXT         -- e.g. 'first_edition' | 'reprint' | 'undated_estimate'
);
CREATE INDEX work_date_entity_idx ON work_date (entity_id);

-- person_derived, including its documented multiline/multi-author cases
-- (§1.1) — a junction, not a single nullable person_id column, so more
-- than one author per work is representable without a schema change.
CREATE TABLE work_author (
  work_author_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  work_id        BIGINT NOT NULL REFERENCES work(entity_id),
  person_id      BIGINT REFERENCES person(entity_id),   -- NULL until resolved
  author_text_raw TEXT NOT NULL,      -- the person_derived line, always kept
  resolution_method TEXT,             -- 'exact_label_match' | 'manual' | ...
  confidence     NUMERIC(3,2),
  UNIQUE (work_id, author_text_raw)
);
CREATE INDEX work_author_work_idx ON work_author (work_id);
CREATE INDEX work_author_person_idx ON work_author (person_id) WHERE person_id IS NOT NULL;

CREATE TABLE work_language_assessment (
  entity_id     BIGINT PRIMARY KEY REFERENCES work(entity_id),
  lang          TEXT NOT NULL,       -- ISO 639-1, e.g. 'de'
  method        TEXT NOT NULL,
  confidence    NUMERIC(4,3)
);

-- ============================================================
-- Place subtype
-- ============================================================
CREATE TABLE place (
  entity_id     BIGINT PRIMARY KEY REFERENCES entity(entity_id)
  -- Deliberately no other columns: entities.csv carries nothing else for
  -- places today (§1.1). Coordinates/country live in place_geocode,
  -- which is sparse (63/2,508) and sometimes ambiguous — see below.
);

-- Reconciliation candidates, not a single lat/lon pair, because the
-- source data itself records unresolved multi-candidate cases
-- (sv14_places_ambiguous.csv). status defaults to 'candidate'; a partial
-- unique index (not a table-level UNIQUE) enforces at most one confirmed
-- geocode per place while still allowing multiple rejected/candidate
-- rows to be kept for audit.
CREATE TABLE place_geocode (
  place_geocode_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id      BIGINT NOT NULL REFERENCES place(entity_id),
  label_en       TEXT,
  lat            NUMERIC(9,6),
  lon            NUMERIC(9,6),
  match_method   TEXT,               -- 'direct' | 'candidate' | ...
  status         reconciliation_status NOT NULL DEFAULT 'candidate',
  sv14_xml_id    TEXT                -- e.g. 'geo-3183356', preserved as-is
);
CREATE INDEX place_geocode_entity_idx ON place_geocode (entity_id);
CREATE UNIQUE INDEX place_geocode_one_confirmed
  ON place_geocode (entity_id) WHERE status = 'confirmed';

-- ============================================================
-- Authority identifiers (§1.5, §4)
-- ============================================================
-- One table for every external identifier system, rather than a wd/viaf/
-- gnd/geonames column per entity type — required because coverage is
-- sparse and type-specific (works get Wikidata, places get GeoNames, no
-- entity type in the current CSVs has more than one system), and because
-- the source data (sv14_places_ambiguous.csv) shows an entity can have
-- more than one *candidate* identifier before reconciliation is final.
CREATE TABLE authority_system (
  system_key    TEXT PRIMARY KEY,     -- 'wikidata' | 'viaf' | 'gnd' | 'geonames' | 'kid' | ...
  label         TEXT NOT NULL,
  url_template  TEXT                  -- e.g. 'https://www.wikidata.org/wiki/{id}'
);

CREATE TABLE authority_identifier (
  authority_identifier_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id       BIGINT NOT NULL REFERENCES entity(entity_id),
  system_key      TEXT NOT NULL REFERENCES authority_system(system_key),
  identifier_value TEXT NOT NULL,
  status          reconciliation_status NOT NULL DEFAULT 'candidate',
  verified_via    TEXT,               -- matches works_wikidata.csv's own column
  verified_at     TIMESTAMPTZ,
  notes           TEXT
);
CREATE INDEX authority_entity_idx ON authority_identifier (entity_id);
-- At most one CONFIRMED identifier per (entity, system) — multiple
-- candidates/rejections stay representable, matching the ambiguous-place
-- evidence in §1.5.
CREATE UNIQUE INDEX authority_one_confirmed
  ON authority_identifier (entity_id, system_key) WHERE status = 'confirmed';

-- ============================================================
-- Derived/inferred person attributes (§1.6) — explicitly NOT register
-- data; every column here traces to a project-run heuristic, not a
-- transcription of the source register.
-- ============================================================
CREATE TABLE person_gender_assessment (
  entity_id       BIGINT PRIMARY KEY REFERENCES person(entity_id),
  gender          gender_value NOT NULL,
  confidence      NUMERIC(3,2),
  has_conflict    BOOLEAN NOT NULL DEFAULT false,
  indicator_count SMALLINT,
  indicators      TEXT,
  explanation     TEXT,
  -- from person_gender_review.csv (§1.6) — merged in, not a second table,
  -- since it's a same-grain subset needing human attention.
  human_reviewed_gender gender_value,
  review_comment  TEXT,
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ
);

CREATE TABLE person_role (
  role_key      TEXT PRIMARY KEY,     -- e.g. 'Akademiker/Lærd'
  label         TEXT NOT NULL
);

CREATE TABLE person_role_assignment (
  entity_id     BIGINT NOT NULL REFERENCES person(entity_id),
  role_key      TEXT NOT NULL REFERENCES person_role(role_key),
  from_work_title BOOLEAN NOT NULL DEFAULT false,   -- kilde_vaerk
  from_description_terms TEXT,                       -- kilde_beskrivelse_termer
  PRIMARY KEY (entity_id, role_key)
);

CREATE TABLE ethnic_adjective (
  adjective_key TEXT PRIMARY KEY,     -- 'svensk', 'holstensk', ...
  label_da      TEXT NOT NULL
);

CREATE TABLE nation_umbrella (
  umbrella_key  TEXT PRIMARY KEY,     -- 'dansk', 'tysk', ...
  label         TEXT NOT NULL,
  place_label_da TEXT,
  notes         TEXT
);

-- M:M, not a parent_umbrella FK on ethnic_adjective — holstensk belongs
-- to both dansk and tysk in the source data (§1.7).
CREATE TABLE nation_umbrella_member (
  umbrella_key  TEXT NOT NULL REFERENCES nation_umbrella(umbrella_key),
  adjective_key TEXT NOT NULL REFERENCES ethnic_adjective(adjective_key),
  PRIMARY KEY (umbrella_key, adjective_key)
);

-- A mention, not a person attribute: referent_hint can be non-subject
-- (§1.6), and a person can carry more than one row (165 persons do).
CREATE TABLE person_nationality_mention (
  mention_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id     BIGINT NOT NULL REFERENCES person(entity_id),
  matched_form  TEXT NOT NULL,
  adjective_key TEXT NOT NULL REFERENCES ethnic_adjective(adjective_key),
  category      TEXT,
  position_index SMALLINT,
  position_type match_position,
  match_type    TEXT,
  referent_hint referent_hint NOT NULL DEFAULT 'subject'
);
CREATE INDEX nat_mention_entity_idx ON person_nationality_mention (entity_id);
-- Only "subject" mentions should feed a nationality facet/filter — a
-- partial index keeps that the cheap, common query path.
CREATE INDEX nat_mention_subject_idx ON person_nationality_mention (entity_id, adjective_key)
  WHERE referent_hint = 'subject';

-- ============================================================
-- Diary text and occurrences (§1.3, §1.2, §1.6)
-- ============================================================
CREATE TABLE diary_page (
  page_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  legacy_pag_id TEXT NOT NULL UNIQUE,   -- 'Pag010067'
  vol           TEXT NOT NULL,
  page          TEXT NOT NULL,          -- kept as text: some are non-numeric
  kb_url        TEXT,                   -- kb_diary_links.csv
  UNIQUE (vol, page)
);

-- diary.csv's actual grain (§1.3): several dated/undated text fragments
-- per page, not one row per page.
CREATE TABLE diary_line (
  diary_line_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  page_id       BIGINT NOT NULL REFERENCES diary_page(page_id),
  entry_date    DATE,           -- only populated for vol VI-VII today
  date_precision TEXT,           -- 'day' | 'month' | 'year' | NULL
  heading       TEXT,
  body_text     TEXT NOT NULL,
  line_order    SMALLINT NOT NULL
);
CREATE INDEX diary_line_page_idx ON diary_line (page_id);
CREATE INDEX diary_line_date_idx ON diary_line (entry_date) WHERE entry_date IS NOT NULL;

-- references.csv: page-grain occurrence, entity_label and vol/page
-- dropped as redundant (§1.2) — both are one join away.
CREATE TABLE entity_occurrence (
  occurrence_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  page_id       BIGINT NOT NULL REFERENCES diary_page(page_id),
  entity_id     BIGINT NOT NULL REFERENCES entity(entity_id),
  seq           SMALLINT,
  UNIQUE (page_id, entity_id)
);
CREATE INDEX occurrence_entity_idx ON entity_occurrence (entity_id);
CREATE INDEX occurrence_page_idx ON entity_occurrence (page_id);

-- ner_page_grounding.csv: finer grain than entity_occurrence — the exact
-- character span of a mention, including recorded negative results
-- (§1.6). Kept as its own table, not folded into entity_occurrence,
-- because the grain and the presence of failed-match rows differ.
CREATE TABLE entity_mention (
  mention_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  page_id       BIGINT NOT NULL REFERENCES diary_page(page_id),
  entity_id     BIGINT NOT NULL REFERENCES entity(entity_id),
  mention_text  TEXT,              -- NULL/empty on a failed match
  mention_start INTEGER,
  mention_end   INTEGER,
  confidence    NUMERIC(4,3) NOT NULL,
  method        TEXT NOT NULL,     -- includes 'no_match'
  human_reviewed BOOLEAN NOT NULL DEFAULT false,
  reviewed_mention_text TEXT,
  reviewed_by   TEXT,
  reviewed_at   TIMESTAMPTZ
);
CREATE INDEX mention_page_idx ON entity_mention (page_id);
CREATE INDEX mention_entity_idx ON entity_mention (entity_id);

-- ============================================================
-- updated_at trigger — keeps entity.updated_at honest without relying
-- on every editorial pathway (SQL client, no-code UI, API) to set it.
-- ============================================================
CREATE OR REPLACE FUNCTION touch_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER entity_touch_updated_at
  BEFORE UPDATE ON entity
  FOR EACH ROW EXECUTE FUNCTION touch_updated_at();
```

### 3.1 Why `BIGINT GENERATED ALWAYS AS IDENTITY`, not UUID

See §8 for the full reasoning; short version: nothing in the evidenced
data volume (16,444 register entities, 69,405 occurrences today) needs
UUIDs' distributed-generation property, sequential bigints keep every
index in this schema smaller and better cache-behaved, and
`legacy_reg_id`/`legacy_pag_id` already give every externally-visible
identifier its stable, human-legible form — the bigint PK never needs to
leak into a URL or a citation.

---

## 4. CSV → PostgreSQL mapping

| Source CSV.column | Destination |
|---|---|
| `entities.entity_id` | `entity.legacy_reg_id` |
| `entities.entity_type` | `entity.entity_type` (drives which subtype row is created) |
| `entities.category_h1` | dropped — 1:1 derivable from `entity_type` (`PERSON-REGISTER`/`VÆRK-REGISTER`/`STED-REGISTER`), redundant |
| `entities.genre_h2`, `.form_h3`, `.subform_h4` | `work_category.h2/h3/h4` (works only) |
| `entities.label` | `entity.label` |
| `entities.description` | `person.description` (persons only — 0% fill elsewhere) |
| `entities.see`, `.see_also` | `entity_cross_reference.to_text_raw` (+ `source_field`) |
| *(label-embedded "– Se også:"/", se:")* | `entity_cross_reference` row with `source_field = 'label_embedded'` — **requires a one-time parse pass during import**, not a straight column copy |
| `entities.year_derived` | `work_date.year`/`.date_text` — **split on `", "` during import**, one row per value (multi-valued today, e.g. `"1847, 1872"`) |
| `entities.date_derived` | `work_date.date_text` |
| `entities.person_derived` | `work_author.author_text_raw` (raw), `.person_id` left NULL pending reconciliation — **including a split step for the documented multiline cases** |
| `references.page_id`, `.entity_id` | `entity_occurrence.page_id` (via `diary_page.legacy_pag_id`), `.entity_id` (via `entity.legacy_reg_id`) |
| `references.entity_label`, `.vol`, `.page` | dropped — redundant (§1.2) |
| `references.seq` | `entity_occurrence.seq` |
| `diary.vol`, `.page` | `diary_page.vol`/`.page` (page created once per distinct pair) |
| `diary.date`, `.month`, `.year` | `diary_line.entry_date` (parsed) + `.date_precision` |
| `diary.heading`, `.text` | `diary_line.heading`, `.body_text` |
| `kb_diary_links.pag_id`, `.kb_url` | `diary_page.legacy_pag_id`, `.kb_url` |
| `works_wikidata.rid`, `.wd` | `authority_identifier.entity_id` (via `legacy_reg_id`), `.system_key='wikidata'`, `.identifier_value` |
| `works_wikidata.verified_via`, `.notes` | `authority_identifier.verified_via`, `.notes` |
| `works_wikidata.image_filename`, `.attribution_note` | **not mapped in this schema** — image/media metadata belongs to a future `media_asset` table, out of scope for the person/work/place brief; flagged in §H |
| `sv14_places_reconciled.entity_id`, `.lat`, `.lon` | `place_geocode.entity_id`, `.lat`/`.lon`, `.status='confirmed'` |
| `sv14_places_reconciled.geonames_url`, `.wiki_url` | parsed into two `authority_identifier` rows (`system_key='geonames'` / `'wikipedia'`) rather than kept as URL columns |
| `sv14_places_reconciled.sv14_xml_id`, `.match_method` | `place_geocode.sv14_xml_id`, `.match_method` |
| `sv14_places_ambiguous.candidates` | one `place_geocode` row per candidate, `status='candidate'` — **requires parsing the `candidates` field's internal structure**, not evidenced in this pass (see §H) |
| `person_gender.*` | `person_gender_assessment` (all columns as named) |
| `person_gender_review.menneskelig_vurdering`, `.kommentar` | `person_gender_assessment.human_reviewed_gender`, `.review_comment` — **matched to the base table by `entity_id`, not imported as a separate table** |
| `person_role.roller` | `person_role_assignment` — **split on `;`**, one row per role, `role_key` values become `person_role` lookup rows |
| `person_role.kilde_vaerk`, `.kilde_beskrivelse_termer` | `person_role_assignment.from_work_title`, `.from_description_terms` |
| `person_ethnic_descriptors.*` | `person_nationality_mention` (all columns as named) |
| `work_languages.*` | `work_language_assessment` (all columns as named) |
| `ner_page_grounding.*`, `_review.*` | `entity_mention` (base + reviewed_* columns merged by `(ref_page_id, entity_id, mention_start)`) |
| `nation_umbrellas_da.umbrella_key/_label/place_label_da/notes` | `nation_umbrella` |
| `nation_umbrellas_da.members` | `nation_umbrella_member` — **split on `;`** |

---

## 5. Data-quality issues to resolve before/during import

1. **`entities.person_derived` is unresolved free text**, sometimes
   multiline (multiple authors). Import can attempt exact-label matching
   against `person.label`/`entity.legacy_reg_id` pairs already parsed
   elsewhere in the mockup build scripts (`build_works_extra.py`'s
   `author_from()`), but per that same script's own documented bug class
   (`person_derived` sometimes equals a *title fragment*, not a real
   author — see `billedkunst-artist-extraction.md`), automatic
   resolution will be wrong for a known subset. Land every row with
   `person_id = NULL` and a `confidence`, and route unresolved/low-
   confidence rows to the review queue (§6) rather than guessing.
2. **`entities.year_derived` is sometimes a list, sometimes a range,
   sometimes free text** — `"1847, 1872"` observed directly; other
   patterns (a hyphenated range, a single year) are likely but not
   exhaustively enumerated by this pass. The import script needs a
   documented parser with a fallback "couldn't parse, kept as
   `date_text`" path, not a silent best-effort regex.
3. **The label-embedded "– Se også:"/", se:" redirect convention**
   (§1.1) must be parsed out of `label` during import for both
   `person` and `work` rows — leaving it in `label` verbatim (as today)
   means `entity.label` mixes "the name" and "an editorial pointer" in
   one string, which breaks both alphabetical sorting (already a known
   issue this session fixed for search-query construction) and any
   no-code table view that sorts/filters on `label` directly.
4. **`sv14_places_ambiguous.candidates`'s internal structure is not
   determined by this pass** — only 1 row exists today, insufficient to
   infer a stable format (JSON? semicolon list? something else). Read it
   directly before writing the import parser (§H).
5. **`person_ethnic_descriptors`'s `referent_hint` values other than
   `'subject'`** must not silently populate a person-facing "nationality"
   facet — the existing mockup already gets this right structurally
   (nation.html only uses "leading position" matches for its facet), but
   a future no-code editing view must not simply expose all
   `person_nationality_mention` rows as if they were equally authoritative
   about the person in question.
6. **No `born`/`died` structured columns exist in `entities.csv` at
   all** — the (1825–1912)-style years currently live only inside
   `label`, parsed client-side by the mockup's JS. `person.born_year`/
   `died_year` in §3's schema are therefore populated by a **parser
   written for this import**, not a column copy; get this parser
   reviewed against edge cases (single-sided dates: `"* 1805"`,
   `"† 1875"`, both already handled in the current mockup's JS and worth
   reusing the same regex rather than re-deriving it) before trusting the
   columns for filtering/sorting.
7. **Entity ID collisions across `entities.csv` and `_v092` snapshot.**
   `data/normalized_v092/entities.csv` also exists in the repo (an older
   snapshot, per the `v0.92-structural-diff.md` doc) — import must target
   the current `data/normalized/` tree only; the `_v092` directory is
   historical, not a second live source, but is easy to import by
   accident given the identical column names.

---

## 6. Editorial workflow for non-technical editors

The schema is designed so a no-code layer (Appsmith/ToolJet/Budibase/
NocoDB/etc.) can be pointed at it with minimal custom logic:

- **Dropdowns for controlled values** come free from the `ENUM` types
  (`gender_value`, `editorial_status`, `reconciliation_status`, …) and
  lookup tables (`work_category`, `person_role`, `ethnic_adjective`,
  `nation_umbrella`, `authority_system`) — every one of these renders as
  a foreign-key picker or enum dropdown in any of the named tools without
  custom widgets.
- **Excel-like editing** works directly against `entity` +
  `person`/`work`/`place` for the core fields, and against the junction
  tables (`work_author`, `person_role_assignment`,
  `person_nationality_mention`) as a nested/related-records grid — the
  same shape a spreadsheet's second tab would take.
- **Review queues are already modeled as data, not UI state**: any row
  with `editorial_status = 'needs_review'`, `authority_identifier.status
  = 'candidate'`, `place_geocode.status = 'candidate'`, or
  `entity_mention.human_reviewed = false AND confidence < <threshold>`
  is a queue — a saved filtered view in the no-code tool, no extra table
  needed.
- **Duplicate detection** is supported at the database level by the
  `pg_trgm` index on `entity.label` (`entity_label_trgm_idx`) — a
  no-code "find similar" feature can run a trigram similarity query
  directly; this does not need to be built into the schema beyond that
  index.
- **Authority-file reconciliation** is a form over `authority_identifier`
  scoped to one `entity_id`: add a candidate row, an editor confirms it
  (`status → 'confirmed'`), the partial unique index prevents two
  confirmed identifiers for the same system existing at once.
- **The database stays authoritative, not the editing tool**, because
  every table the no-code layer touches is a normal Postgres table with
  its own constraints (foreign keys, enum types, partial unique indexes)
  — the no-code tool cannot write a value the schema itself would reject,
  regardless of what validation the tool's UI does or doesn't enforce.
  Row-level audit (`entity.updated_at` via trigger; add
  `reviewed_by`/`reviewed_at` columns as already present on
  `person_gender_assessment` and `entity_mention`) means "who changed
  this and when" is answerable from the database alone, not from the
  no-code tool's own activity log.

---

## 7. Public and research interfaces

The schema does not add anything specific to serve these — deliberately,
per the brief's "do not over-design" instruction — but each is a
straightforward read pattern over the tables already defined:

1. **Scholarly editing/review** — §6, directly.
2. **Public searchable register** — `entity` + subtype tables, filtered
   to `editorial_status = 'published'`; `pg_trgm` powers free-text
   search without a separate search engine at current scale (§8).
3. **Links from entities to diary occurrences** — `entity_occurrence`
   joined to `diary_page`/`diary_line`, exactly the join the current
   mockup's `DIARY_REFS`/`references.csv` relationship already expresses.
4. **Filtering/aggregation by date, nationality, type, genre** —
   `work_date`, `person_nationality_mention` (filtered to
   `referent_hint = 'subject'`), `entity_type`, `work_category` are all
   indexed, ordinary `WHERE`/`GROUP BY` targets.
5. **RDF/Linked Open Data export** — `authority_identifier` is the
   natural `owl:sameAs` source (the mockup's existing `work.html` LOD
   callout already demonstrates the target shape); `entity.legacy_reg_id`
   gives every entity a stable local URI segment.
6. **Power BI / analytical tools** — connect directly to Postgres, or
   build the `star-schema.md` dimension/fact views on top of these tables
   (e.g. a `v_reference_fact` view reshaping `entity_occurrence` +
   `entity` + `diary_page` into the "References-In-Diary-Page" fact
   table shape already specified there) rather than maintaining a
   parallel physical warehouse.

---

## 8. Scale

At today's evidenced volume (16,444 entities, 69,405 occurrences, 11,581
+ 9,569 mention rows) this is a **small** Postgres workload by any
measure — the brief's 100,000–500,000+ projection is plausible once
`entity_mention`-grain data (character-offset mentions across the full
diary corpus, not just the currently-transcribed portion) is imported at
full scale, but even 500k rows in the largest table (`entity_mention`)
is not, by itself, a scale that requires partitioning or denormalization
in Postgres.

- **Bigint identity PKs**, not UUIDs (§3.1) — smaller indexes, better
  insert locality, and nothing here needs offline/distributed ID
  generation.
- **Indexing**: every foreign key used in a join above has a
  corresponding index (`entity_occurrence.entity_id`/`.page_id`,
  `entity_mention.entity_id`/`.page_id`, etc.); the `pg_trgm` index on
  `entity.label` covers fuzzy search without a separate search engine.
  Postgres full-text search (`tsvector`) on `person.description` /
  `diary_line.body_text` is worth adding once free-text search over
  diary prose is an actual product requirement — not included in §3
  because no CSV evidence shows it's needed yet (avoiding
  over-engineering per the brief).
- **Normalize, don't denormalize**, at this scale — every table in §3
  is in at least 3NF; the redundant columns actually found in the source
  (`references.entity_label`/`.vol`/`.page`) are dropped, not preserved
  "for query convenience," because a join on an indexed bigint FK over
  69k rows costs nothing worth trading normalization for.
- **JSONB**: not used anywhere in §3, on purpose. `star-schema.md`
  already flags Work's heterogeneous WEMI-style fields (`opus, incipit,
  adapted_from, ultimate_source, creator_note, …`) as the one place JSONB
  is justified — but **none of those fields appear in the CSVs inspected
  for this document** (`entities.csv` has no such columns). If/when that
  richer Work data lands, add a single `work.extra_attributes JSONB`
  column with a GIN index rather than restructuring this schema; doing
  so now would be speculative.
- **Partitioning**: not justified at any volume discussed above.
  `entity_mention` is the only table that could plausibly reach the
  brief's upper bound (500k+); Postgres handles that as a single table
  with a good index without difficulty. Revisit only if `entity_mention`
  grows past several million rows (the full diary corpus fully NER'd at
  fine grain, repeated per candidate match) — and even then, partition
  by `page_id` range/`vol`, not preemptively.
- **Expected query patterns** the indexing above targets: "everything
  mentioned on page X" (`entity_occurrence.page_id`), "everywhere entity
  Y is mentioned" (`entity_occurrence.entity_id`), "all works by author
  X" (`work_author.person_id`), "all persons with nationality X"
  (`person_nationality_mention` partial index), free-text label search
  (`pg_trgm`).

---

## 9. Import strategy

```
CSV (data/normalized/, data/curated/)
  ↓  1. staging tables (one per CSV, columns as-is, TEXT everywhere,
  │     no constraints) — a faithful, lossless load of exactly what's
  │     in the file, so the raw CSV is always reconstructable from the
  │     database.
  ↓  2. validation / normalization (SQL or a small Python/dbt pass):
  │       - split multi-valued columns (year_derived, person_role.roller,
  │         nation_umbrellas.members)
  │       - parse the label-embedded see/see_also convention
  │       - parse born/died years out of person labels
  │       - resolve work_author.person_id where an exact/high-confidence
  │         match exists; leave NULL otherwise
  │       - de-duplicate references.csv's entity_label/vol/page against
  │         the entity/diary_page tables (drop, don't import, per §1.2)
  │       - merge each *_review.csv into its base table by shared key
  ↓  3. authoritative tables (§3) — only rows that pass §2's validation
  │     land here; a row that fails validation goes to a `staging_error`
  │     table with the reason, not silently dropped and not silently
  │     guessed.
  ↓  4. editorial interface (§6) — no-code tool reads/writes these
  │     tables directly (or through a thin view layer for anything that
  │     needs computed columns, e.g. a "display label" that strips the
  │     redirect convention).
  ↓  5. approved changes — an editor's change is a normal UPDATE/INSERT
  │     against the authoritative tables; editorial_status and the
  │     reviewed_by/reviewed_at columns already present on several
  │     tables record who/when without needing a separate changelog
  │     table for routine edits (see §5's provenance-vs-versioning note).
  ↓  6. PostgreSQL (authoritative, queried by the public/research
        interfaces in §7)
```

**What belongs in step 2 (import-time), not step 5 (editorial):**
mechanical, reversible, evidence-based transforms — splitting a
semicolon list, parsing a well-documented redirect convention, computing
a redundant column away. **What stays editorial:** anything requiring
judgment the CSV itself doesn't resolve — confirming a `work_author`
match, choosing between `place_geocode` candidates, resolving a
`person_gender_assessment` conflict, deciding whether a
`person_nationality_mention` genuinely describes the register subject.
The `_review.csv` files already in the repo are exactly this second
category, already separated out by the existing pipeline — the database
schema keeps that separation (confidence/reviewed_* columns) rather than
collapsing it.

---

## Deliverables

### A. Data inventory

| CSV | Columns | Apparent meaning | Destination |
|---|---|---|---|
| `entities.csv` | entity_id, entity_type, category_h1, genre_h2, form_h3, subform_h4, label, description, see, see_also, year_derived, date_derived, person_derived | Master register, 3 sub-registers in one table | `entity`, `person`, `work`, `place`, `work_category`, `work_date`, `work_author`, `entity_cross_reference` |
| `references.csv` | page_id, entity_id, entity_label, vol, page, seq | Occurrence index (entity × diary page) | `entity_occurrence` |
| `diary.csv` | vol, page, date, month, year, heading, text | Transcribed diary text, line grain | `diary_page`, `diary_line` |
| `kb_diary_links.csv` | pag_id, vol, page, kb_url, source | External facsimile links | `diary_page.kb_url` |
| `works_wikidata.csv` | rid, wd, image_filename, attribution_note, verified_via, notes | Curated Wikidata links for works | `authority_identifier` (image/attribution fields unmapped — §H) |
| `sv14_places_reconciled.csv` | entity_id, label_da, matched_name_en, match_method, lat, lon, sv14_xml_id, geonames_url, wiki_url | Curated place geocoding + authority links | `place_geocode`, `authority_identifier` |
| `sv14_places_ambiguous.csv` | entity_id, label_da, candidates | Unresolved multi-candidate geocodes | `place_geocode` (status='candidate') |
| `person_gender.csv` + `_review.csv` | entity_id, label, nationalitet, koen, confidence, konflikt, antal_indikatorer, indikatorer, forklaring (+ menneskelig_vurdering, kommentar) | Derived gender inference + human review | `person_gender_assessment` |
| `person_role.csv` | entity_id, roller, kilde_vaerk, kilde_beskrivelse_termer | Derived role/occupation tags (multi-valued) | `person_role`, `person_role_assignment` |
| `person_ethnic_descriptors.csv` (+`_review.csv`) | entity_id, label, matched_form, nationality_key, nationality_label, category, position_index, position_type, match_type, referent_hint, description | Nationality-adjective mentions in description text | `ethnic_adjective`, `person_nationality_mention` |
| `work_languages.csv` | entity_id, label, lang, method, confidence | Derived original-language inference | `work_language_assessment` |
| `ner_page_grounding.csv` + `_review.csv` | ref_page_id, vol, page, entity_id, entity_label, mention_text, mention_start, mention_end, confidence, method | Character-offset NER grounding (incl. failed matches) | `entity_mention` |
| `nation_umbrellas_da.csv` | umbrella_key, umbrella_label, place_label_da, members, notes | Nationality-adjective clustering (M:M) | `nation_umbrella`, `nation_umbrella_member` |

### B. Conceptual entity-relationship model

**In plain language:** every register row is an `entity` (person, work,
or place). Persons, works, and places each add their own type-specific
attributes as a 1:1 subtype table. A person can be the (possibly
unresolved) author of many works; a work can have more than one author
candidate. Every entity can carry zero or more external authority
identifiers (Wikidata, GeoNames, …), each independently confirmed or
still a candidate. Every entity can point at other entities (or
unresolved text) as a cross-reference/redirect. Diary text is organized
into pages, each page holding one or more transcribed lines/sections.
Entities occur on diary pages (`entity_occurrence`); some of those
occurrences have also been grounded to an exact text span
(`entity_mention`), sometimes unsuccessfully. Persons additionally carry
project-derived (not source) gender, role, and nationality-mention data,
each with its own confidence/review trail; works carry a derived
language assessment; places carry a (possibly ambiguous) geocode.

```mermaid
erDiagram
    entity ||--o| person : "is-a"
    entity ||--o| work : "is-a"
    entity ||--o| place : "is-a"
    entity ||--o{ entity_cross_reference : "from"
    entity ||--o{ authority_identifier : "has"
    entity ||--o{ entity_occurrence : "occurs on"
    entity ||--o{ entity_mention : "mentioned in"

    work }o--|| work_category : "categorized as"
    work ||--o{ work_date : "dated by"
    work ||--o{ work_author : "authored by"
    work_author }o--o| person : "resolves to"
    work ||--o| work_language_assessment : "assessed"

    place ||--o{ place_geocode : "geocoded by"

    person ||--o| person_gender_assessment : "assessed"
    person ||--o{ person_role_assignment : "has role"
    person_role_assignment }o--|| person_role : "is"
    person ||--o{ person_nationality_mention : "mentioned as"
    person_nationality_mention }o--|| ethnic_adjective : "matches"
    ethnic_adjective ||--o{ nation_umbrella_member : "member of"
    nation_umbrella ||--o{ nation_umbrella_member : "has member"

    diary_page ||--o{ diary_line : "contains"
    diary_page ||--o{ entity_occurrence : "hosts"
    diary_page ||--o{ entity_mention : "hosts"

    entity {
        bigint entity_id PK
        text legacy_reg_id UK
        entity_type entity_type
        text label
        editorial_status editorial_status
    }
    person {
        bigint entity_id PK_FK
        text description
        smallint born_year
        smallint died_year
    }
    work {
        bigint entity_id PK_FK
        bigint category_id FK
    }
    place {
        bigint entity_id PK_FK
    }
    diary_page {
        bigint page_id PK
        text legacy_pag_id UK
        text vol
        text page
    }
    entity_occurrence {
        bigint occurrence_id PK
        bigint page_id FK
        bigint entity_id FK
        smallint seq
    }
    authority_identifier {
        bigint authority_identifier_id PK
        bigint entity_id FK
        text system_key FK
        text identifier_value
        reconciliation_status status
    }
```

### C. Proposed PostgreSQL schema

Full `CREATE TABLE` statements are in §3 above.

### D. Mapping from CSV → PostgreSQL

See §4 above.

### E. Data-quality issues

See §5 above.

### F. Editorial workflow

See §6 above.

### G. Architecture recommendation

**Adopt PostgreSQL with the normalized schema in §3 as the authoritative
data layer, and treat `star-schema.md`'s dimension/fact model as a set of
views (or a scheduled materialization) built on top of it, not a
separate database to keep in sync by hand.**

Reasoning, weighed against `star-schema.md`'s own five candidate
architectures:

- The current CSV-and-Excel-MVP stage (options 1–2 there) has already
  demonstrated the model on real data; the project's own documented next
  step is "Postgres + JSONB is the natural next step if the mockup
  graduates into a longer-lived tool" — this document is that step,
  scoped to what the CSVs evidence today rather than the full WEMI/JSONB
  surface, which isn't yet evidenced (§8).
- A normalized OLTP schema, not a star schema, is the right shape for
  the **editorial** side of this project (scholarly review, authority
  reconciliation, no-code editing) — the star schema is optimized for
  read-heavy analytical aggregation, which is a different, later-stage
  concern this schema doesn't foreclose (§7.6).
- Plain PostgreSQL (option 3), not PostgreSQL+JSONB as the primary shape
  (option 4): every column in §3 traces to an evidenced CSV column with a
  clear type; nothing in the current data is irregular enough to need a
  JSONB escape hatch yet. Add it later, narrowly, per §8, rather than
  defaulting to it now.
- A graph store (option 5) remains, as `star-schema.md` already
  concludes, a post-mockup consideration — the one relationship that
  would most benefit from a graph model (work↔place) has no source data
  yet (§2.3).

### H. Open questions

Decisions the CSVs alone cannot settle — need project-team input:

1. **Is `references.csv` downstream of `ner_page_grounding.csv`, or
   independently maintained?** They share 100% of their page-ID space
   (§1.6) but the CSVs don't prove which is authoritative, or whether
   `entity_occurrence` should eventually be *generated from* confirmed
   `entity_mention` rows rather than imported as its own file.
2. **What is `sv14_places_ambiguous.candidates`'s internal format?**
   Only one row exists today (§5.4) — too little evidence to design its
   parser confidently.
3. **Should `work_derived`/image metadata
   (`works_wikidata.image_filename`, `.attribution_note`) get a `media_asset`
   table now, or wait?** Out of scope for "person/work/place registers"
   as literally briefed, but the data already exists in the curated CSV
   and is currently unmapped (§4).
4. **Does the project want `person.born_year`/`died_year` populated by
   an import-time parser of the label, or should birth/death remain
   purely derived at query time (a view), given no structured source
   column exists for them?** Affects whether they're indexed columns or
   computed.
5. **What is the intended relationship between `data/normalized/` and
   `data/normalized_v092/`?** (§5.7) Confirm the v092 tree should never
   be imported, to avoid an accidental duplicate load.
6. **Who are the actual editorial roles** (`reviewed_by`,
   `imported_by` in §3) — is there already a defined set of project
   contributor identities to constrain these to, or should they stay
   free text for now?
7. **Confirm the target no-code tool** (Appsmith/ToolJet/Budibase/
   NocoDB/Microsoft Lists) before finalizing §6 — the schema in §3 is
   deliberately tool-agnostic (plain FKs/enums/partial indexes), but the
   *view layer* for "display label without the redirect convention" is
   easiest to build once the tool is chosen (native computed column vs.
   Postgres view vs. app-level formula field all work, but only one is
   worth building first).
8. **Confirm the intended relationship to `star-schema.md`'s "Sørens"
   auxiliary registers** (Artist, Sted, Almanak) — that document already
   flags their location/ownership as pending with Søren; this schema's
   `work_author`/`place_geocode` tables are designed to absorb that data
   once it arrives, but cannot be finalized against it sight unseen.
9. **Is the `hca_db` crosswalk in scope for this database, or does the
   verified subset stay a curated CSV in `hca-open-repo`?** See the
   person addendum §10.1 — today `data/curated/breve_person_crosswalk.csv`
   holds one verified row and `hca-open-repo` remains the source of truth,
   so the two answers have not yet diverged in practice. They will.
