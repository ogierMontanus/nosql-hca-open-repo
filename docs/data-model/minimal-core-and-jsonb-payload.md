# Minimal obligatory core + structured JSON payload — with a worked person entry

Status: proposal, responding to the project team's recommendation that
each post carry **a minimal number of obligatory entries**, with
additional information held in **structured JSON**.

Companion to [`postgres-schema-design.md`](postgres-schema-design.md).
This document does not replace its §3 — it re-cuts where the boundary
between columns and payload falls, and says explicitly which tables the
payload must **not** absorb.

All fill rates below were counted from `hca-open-repo`'s
`data/normalized/` on 2026-08-26; the worked example in §5 is one real
register person, with every value traced to a file.

---

## 1. The recommendation already has a home in this project

This is not a new direction. `hca-open-repo`'s
[`wemi-and-relations.md`](https://github.com/ogiermontanus/hca-open-repo/blob/main/docs/data-model/wemi-and-relations.md)
§"Hybrid relational + JSONB design" argues the same position and gives
the reason:

> Two design extremes both lose: a fully normalised 3NF schema leaves
> most rows mostly NULL; a single JSONB blob throws away the columns we
> actually search on.

and, decisively, the limit:

> JSONB cannot answer multi-hop traversal queries efficiently — that is
> exactly what `relation` is for.

So the team's recommendation and the project's own documented design
agree. The work here is drawing the line precisely enough that two
people implementing it independently put the same field on the same side.

---

## 2. The boundary rule

A field belongs in the **JSON payload** when all four hold:

1. **Descriptive, not referential** — nothing joins to it.
2. **Sparse or type-conditional** — it is absent for most rows, or only
   meaningful for one entity type.
3. **No editorial workflow** — nobody reviews, approves or signs off on
   this field individually.
4. **Not a sort key** — no list is ordered by it.

A field must stay a **column or its own table** when any of these hold:

1. **It is a join target** — a foreign key, in either direction.
2. **It is a graph edge** — cross-references, relations, family links.
3. **It carries a review state** — `status`, `verified_via`,
   `reviewed_by`, `confirmed`/`candidate`.
4. **It is a uniqueness or integrity constraint** — the database cannot
   enforce `UNIQUE` or a partial index over a JSON path as cheaply or as
   legibly.
5. **It is the primary sort or range filter** — birth/death years drive
   chronological ordering and range queries.

The fourth "must stay" rule is the one most often lost. The crosswalk
tables from the [person](postgres-schema-design-addendum-hca-db-persons.md)
and [works](postgres-schema-design-addendum-hca-db-works.md) addenda exist
precisely because a mapping is *a claim with a tier, evidence and a
reviewer*. Moving `person_external_map` into a JSON blob would delete the
constraint that stops a 2.8 %-conflict candidate being promoted silently.
**JSON is for description, not for governance.**

---

## 3. What is actually obligatory

Measured over the 10,228 `entity_type='person'` rows:

| Column | Fill | Obligatory? |
|---|---|---|
| `label` | **100.0 %** | **yes** |
| `description` | 92.5 % | no — 762 persons have none |
| label carries birth/death years | 72.3 % | no — 2,833 have none |
| determinate gender assessment | 68.4 % | no — derived, and 3,228 are `Endnu ubestemt` |
| any role assigned | 54.4 % | no |
| any nationality descriptor | 24.6 % | no |
| `see` / `see_also` / `year_derived` / `person_derived` | **0.0 %** | no — work-only columns |

So the obligatory core for a person post is genuinely small: **an
identifier, a type, a label, an editorial status, and a provenance
pointer.** Everything else is optional, and four of the columns the
current wide table carries are never populated for persons at all.

Note what this measurement kills: `description` at 92.5 % looks
mandatory-ish and is not. 762 real register persons would be blocked by a
`NOT NULL` on it.

---

## 4. Schema

```sql
-- ============================================================
-- 4.1  The obligatory core — five columns and a payload
-- ============================================================
CREATE TABLE entity_core (
  entity_id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_type      entity_type NOT NULL,
  label            TEXT NOT NULL CHECK (btrim(label) <> ''),
  editorial_status editorial_status NOT NULL DEFAULT 'published',
  import_batch_id  BIGINT NOT NULL REFERENCES import_batch(batch_id),

  -- Nullable: parser-minted entities have no register id
  -- (parsed-works addendum §2.2), so this cannot be NOT NULL.
  legacy_reg_id    TEXT UNIQUE,

  -- Promoted out of the payload ONLY because they are range-filtered and
  -- sorted on every person list. Nothing else earns a column.
  born_year        SMALLINT,
  died_year        SMALLINT,

  payload          JSONB NOT NULL DEFAULT '{}'::jsonb,

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  CHECK (born_year IS NULL OR died_year IS NULL OR born_year <= died_year),
  CHECK (jsonb_typeof(payload) = 'object')
);

CREATE INDEX entity_core_label_trgm ON entity_core USING gin (label gin_trgm_ops);
CREATE INDEX entity_core_type_idx   ON entity_core (entity_type);
CREATE INDEX entity_core_years_idx  ON entity_core (born_year, died_year);

-- jsonb_path_ops: smaller and faster than the default for the only
-- operator the facets need (@>). Facet queries stay indexed even though
-- the facet values live in the payload.
CREATE INDEX entity_core_payload_idx ON entity_core USING gin (payload jsonb_path_ops);

-- ============================================================
-- 4.2  Payload discipline — a schema, not a junk drawer
-- ============================================================
-- An unvalidated JSONB column becomes unqueryable within a year because
-- three importers spell the same key three ways. Every payload declares
-- its shape version, and the shape is validated on write.
ALTER TABLE entity_core ADD CONSTRAINT payload_has_schema_version
  CHECK (payload ? '_schema' AND jsonb_typeof(payload->'_schema') = 'string');

-- Top-level keys are a closed set per entity type. New sections require a
-- migration — which is the point: it makes adding one a decision.
CREATE OR REPLACE FUNCTION payload_keys_allowed(t entity_type, p JSONB)
RETURNS BOOLEAN LANGUAGE sql IMMUTABLE AS $$
  SELECT NOT EXISTS (
    SELECT 1 FROM jsonb_object_keys(p) AS k
    WHERE k <> ALL (CASE t
      WHEN 'person' THEN ARRAY['_schema','names','life','description',
                               'roles','nationality','gender','affiliations',
                               'source_notes','external_views']
      WHEN 'work'   THEN ARRAY['_schema','titles','creation','genre',
                               'wemi','description','source_notes','external_views']
      WHEN 'place'  THEN ARRAY['_schema','names','geo','typology',
                               'description','source_notes','external_views']
    END)
  );
$$;

ALTER TABLE entity_core ADD CONSTRAINT payload_keys_are_known
  CHECK (payload_keys_allowed(entity_type, payload));

-- ============================================================
-- 4.3  What the payload must NOT absorb (§2)
-- ============================================================
-- These stay exactly as designed in postgres-schema-design.md §3 and the
-- two hca_db addenda. Listed here so the boundary is explicit rather
-- than assumed:
--
--   entity_cross_reference      -- graph edge
--   entity_alias                -- join target, uniqueness constraint
--   authority_identifier        -- review state + one-confirmed partial index
--   person_external_map         -- tier, hold-out evidence, promotion CHECK
--   work_external_map           -- WEMI levels + cross-level CHECK
--   external_person / _work / _publication  -- verbatim source registers
--   letter / letter_participant -- graph edge
--   entity_occurrence           -- 69,405 rows; the index proper
--   entity_mention              -- character spans + review state
--   person_gender_assessment    -- review workflow (human_reviewed_gender)
--
-- A DENORMALIZED COPY of the last one's outcome may appear in the payload
-- for display convenience (§5's `gender` block). The assessment table
-- stays authoritative; the payload copy is a cache and is regenerated,
-- never edited.
```

---

## 5. Worked example — a lengthy person entry

**Edvard Collin (1808–1886)**, chosen because he is the single richest
person on both sides: **767 diary occurrences** (the most of any register
person) and **742 letters** to and from Andersen — the largest
correspondence in the database.

Every value below is real. Sources: `entities.csv` `Reg0048570`;
`references.csv`; `person_gender.csv`; `person_role.csv`;
`ner_page_grounding.csv`; and `hca_db`'s `brevperson` `PersID=218` +
`brev_person` + `brev`.

### 5.1 The row

| Column | Value |
|---|---|
| `entity_id` | `4857` *(surrogate)* |
| `entity_type` | `person` |
| `label` | `Collin, Edvard (1808–1886)` |
| `editorial_status` | `published` |
| `legacy_reg_id` | `Reg0048570` |
| `born_year` | `1808` |
| `died_year` | `1886` |
| `import_batch_id` | `1` |

Eight columns. That is the whole relational footprint of the richest
person in the register.

### 5.2 The payload

```json
{
  "_schema": "person/1.0",

  "names": {
    "family": "Collin",
    "given": "Edvard",
    "display": "Collin, Edvard",
    "sort_key": "collin, edvard"
  },

  "life": {
    "born": { "year": 1808, "date": "1808-11-02", "precision": "day",
              "source": "hca_db.brevperson#218" },
    "died": { "year": 1886, "date": null, "precision": "year",
              "source": "hca_db.brevperson#218",
              "note": "hca_db records 1886-00-00; the register label agrees on the year and neither source gives a day" },
    "home_country": { "value": "Danmark", "source": "hca_db.brevperson#218.Hjemland" }
  },

  "description": {
    "register_da": "Søn af Jonas C. d. Æ., cand. jur. 1831, 1832 Auskultant i Finansdeputationen, 1835 Assessor auscultans, 1840 virkelig Assessor, 1842 Kommitteret, 1848 Direktør i Finansministeriets Departement for Assignations- og Møntvæsen samt Skattesager, Afsked 1864; 1832–1842 Sekretær ved Fonden ad usus publicos, 1851 Medlem af Direktionen for Sparekassen for Kjøbenhavn og Omegn, 1864–1886 administrerende Direktør; Medstifter af Musikforeningen.",
    "source": "entities.csv#Reg0048570.description",
    "external_da": "s. af Jonas Collin, g.m. Henriette C.",
    "external_source": "hca_db.brevperson#218.Biografi"
  },

  "roles": {
    "assigned": ["Embedsmand/Jura/Politik", "Forfatter/Digter", "Handel/Erhverv"],
    "method": "derived",
    "source": "person_role.csv",
    "evidence_terms": ["assessor", "assessor", "direktør", "direktør"],
    "evidence_work_page": "bibliotek.html",
    "stated_title": { "value": "Cand.jur.", "source": "hca_db.brevperson#218.Titel" }
  },

  "gender": {
    "value": "Mandlig",
    "confidence": 0.986,
    "conflict": false,
    "indicator_count": 2,
    "method": "derived",
    "explanation_da": "2 uafhængige indikatorer peger på mandlig.",
    "cache_of": "person_gender_assessment",
    "note": "Display cache. The assessment table is authoritative; this block is regenerated, never edited."
  },

  "nationality": null,

  "source_notes": {
    "occurrence_summary": {
      "diary_pages": 767,
      "by_volume": { "I": 29, "II": 28, "III": 51, "IV": 46, "V": 94,
                     "VI": 134, "VII": 97, "VIII": 124, "IX": 74, "X": 90 },
      "cache_of": "entity_occurrence",
      "note": "Denormalized count for display. Never queried against — entity_occurrence is."
    },
    "ner_grounding": {
      "attempted_pages": 262,
      "cache_of": "entity_mention",
      "note": "Includes recorded no_match rows; a low match rate here is data, not failure."
    },
    "correspondence_summary": {
      "letters": 742,
      "as_recipient": 433,
      "as_sender": 309,
      "span": "1828–1886",
      "partial_dates": 46,
      "by_decade": { "1820s": 8, "1830s": 143, "1840s": 152, "1850s": 89,
                     "1860s": 233, "1870s": 106, "1880s": 4 },
      "cache_of": "letter_participant",
      "note": "Largest correspondence in hca_db. Counts are distinct BrevID."
    }
  },

  "affiliations": [
    { "kind": "institution", "label_da": "Musikforeningen", "role_da": "Medstifter",
      "from_description": true, "resolved_entity_id": null }
  ],

  "external_views": {
    "brevbase_person": "https://andersen.sdu.dk/brevbase/person.html?breve=sendt&pid=218"
  }
}
```

### 5.3 What is deliberately *not* in the payload

This is the half of the example that matters, because it is where a
JSON-first design usually goes wrong.

| Fact about Collin | Where it lives | Why not JSON |
|---|---|---|
| The 767 individual diary occurrences | `entity_occurrence` | The index proper — 69,405 rows project-wide, joined from both directions |
| The 262 NER mentions with character spans | `entity_mention` | Grain is finer than the entity; carries review state |
| The 742 individual letters | `letter` + `letter_participant` | Graph edge; each letter joins to a date and a counterpart |
| **Son of Jonas Collin d.Æ.** → `Reg0048690` | family-relation edge table | A graph edge. Confirmed on both sides: the register says *"Søn af Jonas C. d. Æ."*, `hca_db` says *"s. af Jonas Collin"* |
| **Married to Henriette Oline Collin, f. Thyberg** → `Reg0048650` | family-relation edge table | Reciprocally confirmed: her own register description reads *"g. 1836 m. Edvard C."* |
| The crosswalk to `hca_db` `PersID=218` | `person_external_map` | Carries `match_tier='surname_birthyear_unique'`, `died_year_agrees=true`, `letter_count=742`, and the promotion CHECK |
| Any Wikidata/VIAF id | `authority_identifier` | Review state + the one-confirmed-per-system partial index |

The two family relations are worth dwelling on. In the sources they are
**prose fragments** — `"Søn af Jonas C. d. Æ."` and `"s. af Jonas Collin,
g.m. Henriette C."`. It is tempting to leave them as strings in the
payload, and for the 87 `hca_db` biographies beginning `g.m.` that is
exactly what a JSON-first reading would do. But both resolve to real
register entities, and once resolved they are edges: *"show me everyone
in the Collin household"* is a traversal, and `wemi-and-relations.md`'s
rule applies unchanged — JSONB cannot answer it efficiently.

The payload keeps the **raw prose** (`description.external_da`); the edge
table keeps the **resolved link**. Neither replaces the other, which is
the same propose/verify split the crosswalk tables use.

### 5.4 The crosswalk row that accompanies this entry

```
person_external_map
  entity_id          4857
  register_key       hca_db.brevperson
  external_id        218
  status             candidate          ← not confirmed; awaiting review
  match_tier         surname_birthyear_unique
  died_year_agrees   true               ← hold-out: 1886 = 1886
  nationality_agrees NULL               ← no descriptor on the register side
  letter_count       742                ← review-queue sort key: rank 1
```

Collin is the **top row of the review queue**: the largest
correspondence, a unique surname+birth-year match (22 Collin rows in the
register, exactly one born 1808), and a clean death-year hold-out. He is
still `candidate`, because the tier's measured 2.8 % conflict rate means
the tier does not confirm itself.

---

## 6. Queries the design still has to answer

A minimal core is only safe if the facets survive it.

```sql
-- Persons with a given role (payload containment, GIN-indexed)
SELECT entity_id, label FROM entity_core
WHERE entity_type = 'person'
  AND payload @> '{"roles":{"assigned":["Musiker/Scenekunst"]}}';

-- Chronological range — the reason born_year/died_year are columns
SELECT entity_id, label FROM entity_core
WHERE entity_type = 'person' AND born_year BETWEEN 1800 AND 1810
ORDER BY born_year, label;

-- Facet counts for the gender filter
SELECT payload #>> '{gender,value}' AS koen, count(*)
FROM entity_core WHERE entity_type = 'person'
GROUP BY 1 ORDER BY 2 DESC;
```

The role query is the load-bearing one: it proves a facet can live in the
payload and stay indexed — `EXPLAIN` confirms `@>` is served by
`entity_core_payload_idx` rather than a sequential scan. (With the
`entity_type` predicate also present the planner may prefer the smaller
type index; that is a cost decision on a given table size, not a
limitation of the payload index.) What it cannot do is *rank by evidence quality
or filter on review state* — which is why `person_gender_assessment`
survives as a table and the payload's `gender` block is labelled a cache.

---

## 7. Verification

The DDL in §4 was executed against **PostgreSQL 16.13** on top of
`postgres-schema-design.md` §3 and the three addenda, and the §5 payload
was inserted as a real row and queried back. Behaviour-tested:

| Test | Expected | Result |
|---|---|---|
| Insert the full Collin row + payload | accepted | ✓ |
| Payload without `_schema` | rejected | `payload_has_schema_version` ✓ |
| Payload with an unknown top-level key (`"favourite_colour"`) | rejected | `payload_keys_are_known` ✓ |
| A `work`-only key (`"wemi"`) on a `person` row | rejected | same CHECK ✓ |
| Payload set to a JSON array instead of an object | rejected | `jsonb_typeof` CHECK ✓ |
| `born_year > died_year` | rejected | range CHECK ✓ |
| Empty/whitespace `label` | rejected | `btrim` CHECK ✓ |
| The §5.2 payload parses as valid JSON | yes | ✓ (`json.loads`) |
| Role facet query via `@>` | returns Collin | ✓ |
| A role he does *not* hold | returns nothing | ✓ |
| `EXPLAIN` on the bare `@>` predicate | served by the payload GIN index | ✓ `Bitmap Index Scan on entity_core_payload_idx` |
| Birth-year range query | returns Collin | ✓ |
| Nested reads (`#>>` into `source_notes`, `external_views`) | return 742 and the `pid=218` URL | ✓ |

---

## 8. What this costs, honestly

1. **Payload fields cannot be foreign-keyed.** `affiliations[].resolved_entity_id`
   is a bare integer in JSON — nothing stops it pointing at a deleted
   entity. If institutions become real entities, that array must graduate
   to a table.
2. **No per-field history.** A column change is diffable; a payload
   rewrite is one blob replacing another. If field-level audit is wanted,
   it needs a payload-diff trigger, not JSONB alone.
3. **The closed key set is a migration every time.** That is deliberate —
   but it means "just add a field" is a schema change, and the team should
   agree that is the trade they want.
4. **Denormalized counts go stale.** `occurrence_summary.diary_pages: 767`
   is right today. Every cache block is marked `cache_of` for exactly this
   reason, and they must be regenerated by the same job that loads the
   underlying table, never hand-edited.
5. **`entity_core` vs. the existing `entity`/`person`/`work`/`place`
   tables is an either/or**, not an addition. Adopting this means
   collapsing the subtype tables into the payload and migrating §3 —
   which is §9.

---

## 9. Open questions

1. **Adopt or coexist?** §4's `entity_core` replaces `entity` + the three
   subtype tables. Migrating is a one-time cost; running both is a
   permanent one. The team should pick one, and this document recommends
   replacing — the subtype tables hold 0–2 columns each after the payload
   takes the rest.
2. **Does `description` earn a column after all?** At 92.5 % fill it is
   the strongest payload candidate to promote, and it is what full-text
   search would target. A generated column over `payload->>'description'`
   plus a `tsvector` index would give search without breaking the minimal
   core.
3. **Should facet values be *validated* against the controlled
   vocabularies?** The 9 role keys and 3 gender values exist as lookup
   tables today. A payload can carry a typo that a foreign key would have
   caught; a trigger could check them on write.
4. **One payload schema version per entity type, or per section?**
   `_schema: "person/1.0"` versions the whole object. Section-level
   versioning is more flexible and more bookkeeping.
