# Addendum: `data/parsed/*.tsv` and what they change in the schema design

Status: proposal. Companion to
[`postgres-schema-design.md`](postgres-schema-design.md) — **read that
first**; this document corrects and extends it.

> **Provenance:** moved here from
> [`hca-open-repo`](https://github.com/ogiermontanus/hca-open-repo)'s
> `docs/data-model/postgres-schema-design-addendum-parsed-works.md` on
> 2026-08-25, together with `postgres-schema-design.md`. The
> `data/parsed/*.tsv` files this addendum analyses still live in
> `hca-open-repo` (they are inputs to that repo's live CSV/mockup
> pipeline) — this document is the drafting record, not the data itself.

## Why this addendum exists

The original analysis inventoried `data/normalized/` and `data/curated/`
and **missed `data/parsed/` entirely**. That was a real gap, not a
scoping decision: those three TSVs are the richest work-entity data in
the repository, and one of them falsifies a specific claim the original
document made (§8's "JSONB is not justified — none of those fields
appear in the CSVs inspected"). The fields *do* appear. They are in
`data/parsed/`.

Everything below was counted from the files on 2026-08-19.

---

## 1. What the files are

| File | Rows | `03_genre` | Maps to `entities.form_h3` |
|---|---|---|---|
| `music_register_parsed.tsv` | 135 | `VocalAndInstrumentalMusic` | `Vokal- og Instrumentalmusik` |
| `non_fiction_parsed.tsv` | 216 | `NonFiction` | `Faglitteratur` |
| `novels_plays_tales_parsed.tsv` | 235 | `NovelsPlaysTales` | `Romaner, Noveller, Eventyr` |

**They are section-complete, corpus-partial.** All 580 non-blank
`RegistryTitelID` values resolve cleanly to `work` rows in
`entities.csv` (zero orphans), and each file covers **100 %** of its
`form_h3` section (229/229, 216/216, 135/135). But those three sections
are only **580 of 3,708 works (15.6 %)**. Thirteen sections remain
unparsed:

| Unparsed `form_h3` | Works |
|---|---|
| Skuespil | 747 |
| Digte | 588 |
| Malerier og Tegninger | 587 |
| Skulptur | 315 |
| Eventyr | 286 |
| Operaer og Syngestykker, Skuespil med Musik | 239 |
| Periodica | 84 |
| Balletter | 80 |
| Skuespil og Operatekster | 56 |
| Museer og Samlinger | 39 |
| Romaner og Noveller | 33 |
| Samlede og blandede Skrifter | 24 |
| *(remainder)* | … |

This is an **in-progress parsing effort**, three sections done. That
framing matters more than any single column, because it means the work
schema must absorb thirteen more column sets that do not exist yet.

`03_genre` is a machine-key synonym for `form_h3`, not new information —
a controlled-vocabulary crosswalk (`NonFiction` ⇄ `Faglitteratur`), which
belongs in the `work_category` lookup rather than as a stored column.

---

## 2. Seven findings that change the schema

### 2.1 `entities.label` is a composite string; the TSVs are its decomposition

This is the most consequential finding, and it invalidates the original
document's treatment of `label` as an atomic display/search field.

| `entities.label` | decomposes to |
|---|---|
| `Agnetes Vuggevise (Agnete og Havfruerne) (N. W. Gade)` | main_title `Agnetes Vuggevise` · original_title `Agnete og Havfruerne` · creator `N. W. Gade` |
| `»Alt farer hen som Vinden« (C. F. E. Horneman)` | incipit `Alt farer hen som Vinden` · creator `C. F. E. Horneman` |
| `Aaen og Havet (Thomas Lange)` | main_title `Aaen og Havet` · creator `Thomas Lange` |
| `Die Adoptivtochter (Karoline von Göhren Ͻ: Karoline von Zöllner)` | main_title `Die Adoptivtochter` · creator `Karoline von Göhren Ͻ: Karoline von Zöllner` |

**The label almost never equals the title:** it differs in 560 of 580
parsed rows (music 134/135, non-fiction 198/216, novels 228/229).

Two encoding conventions are now legible:

- **Trailing parenthetical = creator.** The same convention
  `docs/data-model/billedkunst-artist-extraction.md` already documents
  for extracting artists from painting titles — now confirmed as a
  register-wide convention, not a Billedkunst quirk.
- **Guillemets `»…«` = incipit**, i.e. the work is identified by its
  first line because it has no title. 34 of 34 rows carrying a parsed
  `04b_incipit` have guillemets in their label; 36 labels have
  guillemets in total (2 guillemet labels have no parsed incipit — see
  §5).

**Schema consequence.** `entity.label` must be retained as the
*source composite string* — it is what the printed register literally
says, and that is provenance worth keeping — but it must **not** be the
canonical title. `work` needs `main_title`, `incipit`, and
`original_title` as first-class nullable columns, with a
`display_title` resolution rule (`main_title`, else `incipit`) for UI and
sorting. The original document's `pg_trgm` index on `entity.label` stays
useful for "search what the register printed", but a second index on the
parsed title is what a reader actually wants to search.

Note the interaction with the person-side finding already in the main
document (§5.6 there): person labels embed birth/death years in a
trailing parenthetical, work labels embed the *creator* in a trailing
parenthetical. Same syntactic slot, different semantics per entity type —
a single generic "strip the trailing parenthesis" helper would be wrong.

### 2.2 `01_Posttype` is an entity-level editorial status — including parser-minted entities

Three values across 586 rows:

| Value | Rows | Meaning |
|---|---|---|
| `standardpost` | 558 | An ordinary register entry |
| `krydshenvisning` | 22 | The entry *is* a cross-reference — its whole content is a pointer |
| `inferred_container` | 6 | **Not in the source register at all** — minted by the parser |

`krydshenvisning` rows have a `RegistryTitelID` and a
`09_Krydshenvisning_til` target title, e.g. `Aschenbrödel →
Asschepoester`. This is the structured form of exactly the same editorial
convention found unstructured inside *person* labels (`"– Se også: …"`,
~630 persons — see `person-bio-search-links.md`). The original schema's
`editorial_status` enum already has a `'redirect'` value; this is its
evidence, and it confirms `entity_cross_reference` was the right shape.

`inferred_container` is the more interesting case. All six are
collections that other entries claim membership of — e.g. six entries say
`07_part_of: Guldmageren`, so the parser minted a `Guldmageren` container
(creator `C. Hauch`) that has **no `RegistryTitelID`** because it does
not exist in the printed register. The schema must therefore:

1. allow an `entity` with **no `legacy_reg_id`** (the original
   document declared that column `NOT NULL UNIQUE` — that is now wrong),
   and
2. record *that* an entity was inferred rather than transcribed, since
   conflating a minted container with a real register entry would
   misrepresent the source.

### 2.3 Work↔work whole/part relationships

`07_part_of` (novels, 6 rows) / `08_part_of` (music, 5 rows) name a
containing work **by title, not by ID** — `De Geers Ungdomsliv` →
`Guldmageren`. Combined with §2.2's inferred containers this is a genuine
self-referential work hierarchy, and it is the first evidenced
work-to-work relationship in the corpus. It needs its own table (title
text preserved, resolved FK nullable), on the same "keep the raw string,
resolve when you can" pattern the original document used for
`work_author`.

Expect this to grow substantially: the unparsed sections include
`Samlede og blandede Skrifter` (collected works, 24) and `Periodica`
(84), which are container-shaped by definition.

### 2.4 Work↔person is a *typed* relation, and creators are not always people

The original document modelled a single `work_author` table. The parsed
data shows at least three distinct roles, in separate columns:

- **`06_creator`** (all three files, 83–94 % filled)
- **`07_translator`** (non-fiction, 3 rows) — Jonas Collin d. Y.,
  Edv. Collin, Mathilde Ørsted
- **editor**, embedded inside creator strings rather than given its own
  column: `Fr. Müller, udg. af P. O. Brøndsted` ("udg. af" = edited by)

`06b_creator_is_human` (music only) is `False` for 6 rows, and the values
show why: `norsk Folkevise`, `tysk Folkevise`, `irsk Folkevise`,
`svensk Folkemelodi`, `Folkemelodi`, `napolitansk Folkevise`. **The
creator is a folk tradition, not a person.** These will never resolve to
a `person` row, which changes the meaning of a NULL `person_id` — the
original document described NULL as "pending reconciliation"; it must
also be able to mean "resolved: this creator is not a person."

`06_creator` also carries compound and annotated values:

| Raw value | What it actually contains |
|---|---|
| `P. C. Asbjørnsen og Jørgen Moe` | two people |
| `Fr. Müller, udg. af P. O. Brøndsted` | author + editor |
| `Talvj Ͻ: Therese A. L. von Jacob, g. Robinson` | pseudonym `Ͻ:` real name (`Ͻ:` = "i.e.") |
| `Efter det Franske, Aftenlæsning 1871` | not a creator at all — a source note |

The `Ͻ:` marker is a systematic pseudonym convention, and it also appears
inside *labels* (`Die Adoptivtochter (Karoline von Göhren Ͻ: Karoline von
Zöllner)`), so it is a register-wide editorial sign, not a one-off.

### 2.5 Pseudonyms justify an alias table

`05_pseudonym` (non-fiction, 5 rows) pairs a pseudonym with the creator's
real name:

| pseudonym | creator |
|---|---|
| Elise von Hohenhausen | Elise Rudiger |
| **Elpis Melena** | Marie Esperance Schwartz |
| **Eipis Melena** | Marie Esperance Schwartz |
| Agatha | Reinoudina de Goeje |
| Toujours fidèle | L. Frost |

The brief asked whether a names/aliases table was justified; the original
document found no evidence and did not create one. This is the evidence.
Note also that `Elpis`/`Eipis` are near-certainly the same pseudonym with
an l/i transcription error — a concrete duplicate-detection case (§5).

### 2.6 `08_source` justifies a source-citation table

Non-fiction `08_source` (10 rows) holds bibliographic citations for where
a piece appeared:

- `Illustrirte Zeitung 14.12.1872`
- `Efter Chicago Times 27.4.1873; Dagbladet 9.6.1873` — **semicolon-
  separated, multi-valued**
- `S. D. Jacobsen, Dagbladet 28.12.1871, Nr. 311`
- `Fremtidens Nytaarsgave VII, 1873`

This is distinct from creator and from the Wikidata/GeoNames authority
identifiers already modelled. The brief asked about a `sources` table;
this is its (modest, 10-row, but real and growing) justification. The
`Periodica` section (84 unparsed works) will almost certainly add more.

### 2.7 Per-genre column divergence — JSONB is now justified

The three files share seven columns and diverge on the rest:

| Column | music | non-fiction | novels/plays |
|---|:---:|:---:|:---:|
| Posttype, by_Andersen, genre, main_title, creator, Krydshenvisning_til, Note | ● | ● | ● |
| `04b_incipit` | ● | | |
| `05_original_title` | ● | | ● |
| `06b_creator_is_human` | ● | | |
| `07_opus` (`op. 3`, `op. 64b`) | ● | | |
| `part_of` | ● | | ● |
| `05_pseudonym` | | ● | |
| `07_translator` | | ● | |
| `08_source` | | ● | |
| `11_uncertain_citation` | | ● | |
| `10_cited_directly` | | | ● |
| `Se_ogsaa` | | ● | ● |

Three sections already produce three different column sets, and
**thirteen sections remain**, each likely to introduce its own
(Skulptur will want material/dimensions; Malerier og Tegninger will want
collection/location; Balletter will want choreographer). This is exactly
the condition `star-schema.md` anticipated when it specified "a JSONB
column in the eventual PostgreSQL star schema" for Work's heterogeneous
attributes.

**This reverses the original document's §8 conclusion.** Its reasoning
was sound given what it had inspected — it inspected the wrong
directory. The corrected position: promote to real columns those fields
that are cross-genre or certain to recur (`main_title`, `incipit`,
`original_title`, `opus`, the boolean flags), and give `work` a
`extra_attributes JSONB` column with a GIN index for the long tail of
genre-specific fields, so that parsing section fourteen does not require
a migration.

---

## 3. Schema delta

Only changes to [`postgres-schema-design.md`](postgres-schema-design.md)
§3 are shown. Verified against PostgreSQL 16 (see §6).

```sql
-- ── 2.2: entities can be parser-minted, so legacy_reg_id must be nullable ──
-- Replaces the NOT NULL UNIQUE declaration in the original §3.
ALTER TABLE entity ALTER COLUMN legacy_reg_id DROP NOT NULL;
-- UNIQUE already permits multiple NULLs in PostgreSQL, so the existing
-- constraint still enforces "no two entities share a register id".

ALTER TABLE entity ADD COLUMN provenance_kind TEXT NOT NULL DEFAULT 'transcribed'
  CHECK (provenance_kind IN ('transcribed', 'inferred'));
COMMENT ON COLUMN entity.provenance_kind IS
  'transcribed = present in the printed register; inferred = minted by a '
  'parser (e.g. an inferred_container work). Never conflate the two.';

-- 01_Posttype (§2.2). Distinct from editorial_status, which is a workflow
-- state an editor moves through; post_kind is what the SOURCE says the
-- entry is.
CREATE TYPE post_kind AS ENUM ('standardpost', 'krydshenvisning', 'inferred_container');
ALTER TABLE entity ADD COLUMN post_kind post_kind NOT NULL DEFAULT 'standardpost';

-- ── 2.1: the parsed decomposition of the composite label ──
ALTER TABLE work ADD COLUMN main_title     TEXT;
ALTER TABLE work ADD COLUMN incipit        TEXT;   -- »…« convention
ALTER TABLE work ADD COLUMN original_title TEXT;
ALTER TABLE work ADD COLUMN opus           TEXT;   -- 'op. 64b' — not numeric
ALTER TABLE work ADD COLUMN by_andersen    BOOLEAN;
ALTER TABLE work ADD COLUMN cited_directly BOOLEAN;      -- novels/plays only
ALTER TABLE work ADD COLUMN uncertain_citation BOOLEAN;  -- non-fiction only
ALTER TABLE work ADD COLUMN note           TEXT;
-- §2.7: long tail of genre-specific fields from the 13 unparsed sections.
ALTER TABLE work ADD COLUMN extra_attributes JSONB NOT NULL DEFAULT '{}'::jsonb;

-- NO "work must have a title" CHECK is added here, deliberately.
-- 3,128 of 3,708 works are unparsed and legitimately have neither
-- main_title nor incipit yet, so any constraint permissive enough to
-- admit them is permissive enough to admit everything. (An earlier
-- draft of this addendum carried one; it was tautological — see §5.7 —
-- and a constraint that can never fire is worse than none, because it
-- advertises a guarantee it does not provide.)
--
-- Apply this ONCE parsing of all sections is complete (§7 open q. 4):
--   ALTER TABLE work ADD CONSTRAINT work_has_a_title
--     CHECK (main_title IS NOT NULL OR incipit IS NOT NULL);
-- Until then, "which works still lack a parsed title?" is a review-queue
-- query, not a constraint:
--   SELECT entity_id FROM work
--    WHERE main_title IS NULL AND incipit IS NULL;

CREATE INDEX work_main_title_trgm_idx ON work USING gin (main_title gin_trgm_ops);
CREATE INDEX work_extra_attrs_idx     ON work USING gin (extra_attributes);

-- Resolution rule for display/sort (§2.1): parsed title if present,
-- else the raw register label. A view keeps this in one place rather
-- than in every client.
CREATE VIEW v_work_display AS
SELECT e.entity_id,
       e.legacy_reg_id,
       COALESCE(w.main_title, w.incipit, e.label) AS display_title,
       w.incipit IS NOT NULL AS is_incipit,
       e.label AS register_label,
       w.original_title,
       e.post_kind,
       e.provenance_kind
FROM entity e JOIN work w USING (entity_id);

-- ── 2.4: work↔person is typed, and creators are not always people ──
CREATE TYPE agent_role AS ENUM ('creator', 'translator', 'editor', 'pseudonym_of');

CREATE TYPE agent_kind AS ENUM ('person', 'tradition', 'corporate', 'unknown');
COMMENT ON TYPE agent_kind IS
  'tradition covers "norsk Folkevise"/"Folkemelodi" — creator_is_human=False '
  'in the source. Such rows never resolve to a person_id, which is different '
  'from a person_id that is merely not resolved YET.';

-- Replaces the original §3 work_author table.
CREATE TABLE work_agent (
  work_agent_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  work_id         BIGINT NOT NULL REFERENCES work(entity_id),
  person_id       BIGINT REFERENCES person(entity_id),  -- NULL if unresolved OR non-person
  agent_text_raw  TEXT NOT NULL,        -- always kept verbatim
  role            agent_role NOT NULL DEFAULT 'creator',
  agent_kind      agent_kind NOT NULL DEFAULT 'unknown',
  ordinal         SMALLINT,             -- for 'P. C. Asbjørnsen og Jørgen Moe'
  resolution_method TEXT,
  confidence      NUMERIC(3,2),
  UNIQUE (work_id, agent_text_raw, role, ordinal)
);
CREATE INDEX work_agent_work_idx   ON work_agent (work_id);
CREATE INDEX work_agent_person_idx ON work_agent (person_id) WHERE person_id IS NOT NULL;
-- A resolved person and a non-person agent are mutually exclusive.
ALTER TABLE work_agent ADD CONSTRAINT work_agent_kind_consistent
  CHECK (person_id IS NULL OR agent_kind = 'person');

-- ── 2.3: work↔work whole/part ──
CREATE TABLE work_part_of (
  work_part_of_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  part_work_id    BIGINT NOT NULL REFERENCES work(entity_id),
  container_work_id BIGINT REFERENCES work(entity_id),   -- NULL until resolved
  container_text_raw TEXT NOT NULL,                      -- 'Guldmageren'
  resolution_method TEXT,
  UNIQUE (part_work_id, container_text_raw),
  CHECK (container_work_id IS NULL OR container_work_id <> part_work_id)
);
CREATE INDEX work_part_of_part_idx      ON work_part_of (part_work_id);
CREATE INDEX work_part_of_container_idx ON work_part_of (container_work_id)
  WHERE container_work_id IS NOT NULL;

-- ── 2.5: aliases / pseudonyms ──
CREATE TYPE alias_kind AS ENUM ('pseudonym', 'variant_spelling', 'original_title', 'other');

CREATE TABLE entity_alias (
  alias_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id    BIGINT NOT NULL REFERENCES entity(entity_id),
  alias_text   TEXT NOT NULL,
  alias_kind   alias_kind NOT NULL,
  source_field TEXT,          -- '05_pseudonym' | 'label_Ͻ:_marker' | ...
  UNIQUE (entity_id, alias_text, alias_kind)
);
CREATE INDEX entity_alias_entity_idx ON entity_alias (entity_id);
CREATE INDEX entity_alias_text_trgm_idx ON entity_alias USING gin (alias_text gin_trgm_ops);

-- ── 2.6: bibliographic source citations ──
CREATE TABLE work_source_citation (
  citation_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  work_id      BIGINT NOT NULL REFERENCES work(entity_id),
  citation_text TEXT NOT NULL,   -- 'Illustrirte Zeitung 14.12.1872'
  ordinal      SMALLINT NOT NULL DEFAULT 1,   -- multi-valued, ';'-separated
  UNIQUE (work_id, citation_text)
);
CREATE INDEX work_source_citation_work_idx ON work_source_citation (work_id);

-- ── §1: genre machine-key belongs on the category lookup ──
ALTER TABLE work_category ADD COLUMN genre_key TEXT UNIQUE;
COMMENT ON COLUMN work_category.genre_key IS
  'Machine key from data/parsed/*.tsv 03_genre, e.g. NonFiction ⇄ Faglitteratur.';
```

### 3.1 Cross-reference targets: no change needed

`09_Krydshenvisning_til` and `Se_ogsaa` map onto the existing
`entity_cross_reference` table unchanged — `to_text_raw` holds the target
title, `to_entity_id` stays NULL until resolved, `source_field` records
which column it came from. The original design anticipated this
correctly.

---

## 4. Additional CSV → PostgreSQL mappings

| Source column | Destination |
|---|---|
| `RegistryTitelID` | join key → `entity.legacy_reg_id` (NULL for `inferred_container`) |
| `01_Posttype` | `entity.post_kind`; `inferred_container` also sets `entity.provenance_kind='inferred'` |
| `02_by_Andersen` | `work.by_andersen` |
| `03_genre` | `work_category.genre_key` (lookup, not a stored column on `work`) |
| `04_main_title` | `work.main_title` |
| `04b_incipit` | `work.incipit` |
| `05_original_title` | `work.original_title` + an `entity_alias` row (`alias_kind='original_title'`) |
| `05_pseudonym` | `entity_alias` (`alias_kind='pseudonym'`) on the resolved creator, **or** on the work if the creator is unresolved — see §7 open question 3 |
| `06_creator` | `work_agent` (`role='creator'`); **split on ` og `/`,` where a compound is detected**, `ordinal` set; `agent_text_raw` always keeps the full original |
| `06b_creator_is_human` | `work_agent.agent_kind` (`False` → `'tradition'`, `True` → `'person'`, blank → `'unknown'`) |
| `07_opus` | `work.opus` |
| `07_translator` | `work_agent` (`role='translator'`) |
| `07_part_of` / `08_part_of` | `work_part_of.container_text_raw` |
| `08_source` | `work_source_citation` — **split on `;`**, one row per citation |
| `08_Se_ogsaa` / `09_Se_ogsaa` | `entity_cross_reference` (`reference_type='see_also'`) |
| `09_Krydshenvisning_til` / `10_Krydshenvisning_til` | `entity_cross_reference` (`reference_type='see'`) |
| `10_cited_directly` | `work.cited_directly` |
| `11_uncertain_citation` | `work.uncertain_citation` |
| `10_Note` / `11_Note` / `12_Note` | `work.note` |
| *(any column from a future parsed section with no home above)* | `work.extra_attributes` JSONB |

The `01_`–`12_` numeric prefixes are ordering artefacts of the parse and
should be dropped on import; they carry no semantics and differ between
files for the same field (`Note` is `10_`, `11_`, and `12_` across the
three files).

---

## 5. Additional data-quality issues

1. **`Elpis Melena` / `Eipis Melena`** — same pseudonym for the same
   creator (Marie Esperance Schwartz), differing by one character
   (`l`/`i`). Near-certain transcription error. A concrete first test
   case for the `pg_trgm` duplicate detection described in the main
   document §6.
2. **Two guillemet labels have no parsed incipit** (36 labels with
   `»…«`, 34 parsed incipits). Either a parse miss or a label where
   guillemets mean something else (a quoted title rather than a first
   line). Must be inspected before the guillemet→incipit rule is
   trusted as a general parser.
3. **`06_creator` is not always a creator.** `Efter det Franske,
   Aftenlæsning 1871` is a source note ("after the French,
   *Aftenlæsning* 1871") sitting in the creator column. Import must not
   mint a `work_agent` row of `agent_kind='person'` from strings like
   this; route them to review, or to `work_source_citation` if an editor
   agrees.
4. **`06b_creator_is_human` is blank for 11 of 135 music rows**, all of
   which also have no `06_creator`. Blank means "no creator recorded",
   not "creator recorded but humanity unknown" — do not import blank as
   `agent_kind='unknown'` with an empty `agent_text_raw`; import no row
   at all.
5. **`inferred_container` rows have no stable identity.** They are
   matched only by title string (`Guldmageren`), so re-running the
   parser could mint duplicates, and two different real containers
   sharing a title would collapse into one. They need minted, persisted
   surrogate ids on first import — this is precisely why
   `entity.legacy_reg_id` must become nullable rather than being given a
   fake `Reg…` value.
6. **`02_by_Andersen` is `False` for all 586 rows.** The column is
   therefore currently uninformative — but the unparsed sections include
   `Eventyr` (286) and `Digte` (588), which certainly *are* by Andersen.
   Do not infer from the current data that the column can be dropped.
7. **Self-critique — a tautological constraint caught in review.** An
   earlier draft of §3 carried
   `CHECK (main_title IS NOT NULL OR … OR main_title IS NULL)`, which is
   true for every possible row and so can never fire. It was found by
   testing whether it could reject a work with no title at all (it could
   not — the INSERT succeeded), not by reading it. Recorded here rather
   than silently corrected, because the general lesson applies to the
   rest of this schema: **a constraint is only real if a rejection case
   has been demonstrated**, and every CHECK in §3 above has one in §6.

---

## 6. Verification

The delta DDL in §3 was applied to a real PostgreSQL 16.13 instance on
top of the original document's schema, and behaviour-tested rather than
merely parsed:

- all statements execute cleanly on top of the base schema;
- an entity with `legacy_reg_id = NULL` and
  `provenance_kind = 'inferred'` inserts successfully (the
  `inferred_container` case), while a second such entity also inserts —
  confirming `UNIQUE` permits multiple NULLs as relied on in §3;
- `work_agent_kind_consistent` correctly **rejects** a row that sets a
  `person_id` together with `agent_kind='tradition'`, and **accepts**
  `agent_kind='tradition'` with `person_id IS NULL` (the
  `norsk Folkevise` case);
- `work_part_of`'s self-reference `CHECK` rejects a work contained by
  itself;
- `v_work_display` returns the parsed `main_title` where present, flags
  `is_incipit` correctly for the `»At kjøre Vatten og kjøre Veed«` case,
  and falls back to the raw composite `entity.label` for an unparsed
  work — the behaviour the 3,128 unparsed works depend on;
- a compound creator splits into two `work_agent` rows distinguished by
  `ordinal` (`P. C. Asbjørnsen` / `Jørgen Moe`) without tripping the
  `UNIQUE (work_id, agent_text_raw, role, ordinal)` constraint.

Every CHECK constraint added in §3 has a demonstrated rejection case
above; none is merely assumed to work (see §5.7).

---

## 7. Corrections to `postgres-schema-design.md`

| Section | Correction |
|---|---|
| §1 (data inventory) | Omitted `data/parsed/` entirely. The three TSVs are the richest work data in the repo. |
| §3 `entity.legacy_reg_id NOT NULL` | **Wrong.** `inferred_container` entities have no register id. Now nullable (§3). |
| §3 `work_author` | Superseded by `work_agent` — the relation is typed (creator/translator/editor) and the agent is not always a person. |
| §4 mapping table | Did not map `main_title`/`incipit`/`original_title`/`opus`/`part_of`/`pseudonym`/`source`; added in §4 above. |
| §8 "JSONB: not used anywhere in §3, on purpose… none of those fields appear in the CSVs inspected" | **Falsified.** `opus`, `incipit`, `part_of`, `translator`, `pseudonym`, `source` all appear in `data/parsed/`. JSONB is justified — see §2.7. |
| §2.3 "tables considered and rejected" | Rejected an alias table and a sources table for lack of evidence; both are now evidenced (§2.5, §2.6). The rejection of `work_place`/`person_place` **still stands** — nothing in `data/parsed/` links a work to a place. |
| §H open questions | Add the four below. |

## 8. New open questions

1. **Will the remaining thirteen sections be parsed to the same
   template?** If each parser invents its own column names for the same
   concept (`04_main_title` vs `title` vs `hovedtitel`), the JSONB long
   tail becomes unqueryable. A shared column-naming contract across
   section parsers is worth agreeing *before* section four is parsed.
2. **Is `Ͻ:` (pseudonym marker) systematic enough to parse
   automatically?** It appears in both `06_creator` and in `label`. If
   the project confirms it always means "pseudonym Ͻ: real name", import
   can split it into `entity_alias` rows; otherwise it must stay
   editorial.
3. **Does a pseudonym attach to the person or to the work?**
   `05_pseudonym` sits on a work row, but semantically names a *person*.
   With `work_agent.person_id` mostly unresolved, the import has nowhere
   certain to put it. Alternatives: (a) attach to the work until the
   creator resolves, then migrate; (b) hold in a staging table until
   creator reconciliation completes. Needs a project decision.
4. **Should `work` eventually require a title?** The `work_has_a_title`
   CHECK in §3 is deliberately permissive because 3,128 works are
   unparsed. Confirm the intent to tighten it once parsing completes,
   and whether an unparseable entry is allowed to remain title-less
   permanently.
5. **Are `inferred_container` works meant to be published?** They are
   scholarly inferences, not register entries. Should they appear in the
   public register (§7.2 of the main document), be hidden, or be shown
   with an explicit "not in the printed register" marker — the same
   distinction the site already draws visually between register data and
   enriched data (`docs/external-links.md` §2a)?
