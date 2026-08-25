# Addendum: `hca_db` persons and the crosswalk to the `hca-open-repo` register

Status: proposal. Companion to
[`postgres-schema-design.md`](postgres-schema-design.md) — **read that
first**; this document extends it with a second person source and the
mapping tables that tie the two together.

> **Scope of this addendum.** It covers the **person axis only**. The
> works axis (`bibliografi`, `vaerktitler`, and the Index A/B/C authority
> model in `hca_db_export/docs/authority-model.md`) is a separate,
> larger problem and is deliberately left to a future addendum; §8 notes
> where the two axes touch.

> **Governing principle, as instructed:** `hca-open-repo` is the
> **basis** — for structure and for content. `hca_db` is an *external
> authority to be mapped against*, not a second master. Where the two
> disagree about a person's identity, dates, nationality or role, the
> `hca-open-repo` register row is the one the schema treats as the entity;
> the `hca_db` value is stored as a linked, attributed claim, never
> written over the register. No `hca_db` row creates an `entity` row (§4).

Everything below was counted directly from
`hca_db_export/hca_db.sql` (phpMyAdmin dump, MySQL 5.7, generated
2023-06-29, 64 MB, 80 tables) and from `hca-open-repo`'s
`data/normalized/` CSVs on 2026-08-25. No figure is recalled or
estimated; §7 states the method so every number can be re-derived.

---

## 1. Why this addendum exists

Two things changed since `postgres-schema-design.md` was written.

**First, the correspondence data arrived in relational form.**
`hca-open-repo`'s [`docs/data-model/correspondence-integration.md`](https://github.com/ogiermontanus/hca-open-repo/blob/main/docs/data-model/correspondence-integration.md)
analysed `BrevBasen.csv`, a flat 61-column `;`-delimited export, and
closed with §10 "On the offered SQL dump": the dump would earn its keep
only if it exposed referential integrity that would catch orphan rows
structurally, *tables not flattened into the export*, or a live schema.
**All three conditions are met.** The dump is not a re-delivery of the
CSV; it is the database the CSV was a join over.

**Second, the dump contains two person registers, not one.** That is the
single most important structural fact in this document and it is
invisible from the flat export.

**What this changes for `postgres-schema-design.md`:** nothing in §3's
existing tables is wrong. The delta is additive — one namespaced external
person register, one verified crosswalk table, and one letter/relation
layer (§5).

---

## 2. What is actually in `hca_db` on the person axis

### 2.1 Two disjoint person registers

| Table | Rows | What it actually holds |
|---|---|---|
| `brevperson` | **2,018** | The **correspondence register**: Andersen's historical correspondents, 1820s–1870s. This is the table that matters here. |
| `person` | **676** | A **modern contributor register**: conference speakers, translators, bibliography authors and editors (`Priscilla Meyer`, `Sven Hakon Rossel`, `Ejnar Askgaard`, …), with `PersonEmail`/`PersonURL`/conference-registration fields hanging off it via `personadresser` (662) and `personkonference` (72). It also carries ~60 historical figures used as bibliography authors. |

They are **not two views of one register**. Normalizing names on both
sides (diacritics stripped, embedded HTML removed, surname + first 12
characters of given name) gives **2,000 distinct names in `brevperson`,
581 in `person`, and 12 in common** — Orla Lehmann, Carsten Hauch, Carl
Bagger, Meïr Aron Goldschmidt, J. L. Heiberg, D. G. Monrad and six
others, i.e. exactly the handful of 19th-century figures who are both
Andersen's correspondents and authors of works catalogued in the
bibliography.

Practical consequence: **`person` is out of scope for the register
crosswalk.** Mapping living scholars onto a printed 19th-century diary
index is a category error. `person` belongs to the bibliography/Index A
problem (`bibliografi_person`, 209 rows, `RelationType ENUM('forfatter',
'redaktør','tilegnet','udgiver','oversætter')`) and should be modelled
there. Everything below concerns `brevperson`.

### 2.2 `brevperson` has two candidate keys, and the FK targets the wrong-looking one

```sql
CREATE TABLE `brevperson` (
  `ID`       int(11) NOT NULL DEFAULT '0',   -- surrogate; values ~18,500–20,535
  `PersID`   int(11) NOT NULL,               -- public key; values 1–20,535, sparse
  `MusID`    int(11) NOT NULL DEFAULT '0',   -- museum id; NON-ZERO ON ONLY 5 ROWS
  `Fornavn`  varchar(255), `Efternavn` varchar(100), `Titel` varchar(100),
  `Foedselsdato` date, `Doedsdato` date,
  `Hjemland` enum(... 96 values ...) NOT NULL DEFAULT 'ukendt',
  `Biografi` varchar(255), `Relationer` varchar(255),
  `Timestamp` timestamp
) ENGINE=MyISAM DEFAULT CHARSET=latin1;
```

Both `ID` and `PersID` are unique across the 2,018 rows. The join table
`brev_person` (27,637 rows) points at **`PersID`, not `ID`**: of its
1,921 distinct `PersonID` values, **1,919 resolve against `PersID` and 1
against `ID`** — the two leftovers are `PersonID='0'` and `PersonID='40'`
(4 relation rows total), orphans with no person behind them.

`PersID` is also the **public identifier**: it is the `pid=` in
`https://andersen.sdu.dk/brevbase/person.html?breve=sendt&pid={PersID}`,
independently confirmed in `correspondence-integration.md` §3 and
re-confirmed here (`PersID=151` → Fornavn `Fredrika`, Efternavn `Bremer`,
1801-08-17 – 1865-12-31, Hjemland `Sverige`).

**So `PersID` is the identifier to carry, and it is the join key.** `ID`
is an internal artefact with no external meaning; `MusID` is effectively
unpopulated (5 non-zero rows out of 2,018) and must not be modelled as a
column. `ENGINE=MyISAM` means the dump declares **no foreign keys at
all** — every referential guarantee above is one this import has to
enforce, not one it inherits.

### 2.3 Field coverage — and the three fields the flat export undersold

| Column | Non-empty | % | Notes |
|---|---|---|---|
| `Efternavn` | 1,970 | 97.6 % | 48 rows have none — institutions and single-name figures |
| `Fornavn` | 1,943 | 96.3 % | carries embedded HTML (`<acronym title="Hans Christian">H.C.</acronym>` for `PersID=1`) |
| `Foedselsdato` | 1,693 | 83.9 % | `0000-00-00` = unknown; `1803-00-00` = year only |
| `Doedsdato` | 1,675 | 83.0 % | same convention |
| `Hjemland` | 1,875 | 92.9 % | 96-value ENUM; `Danmark` 1,140, `Tyskland` 280, `Sverige` 109, `England` 96. 139 blank + 4 literal `ukendt` — both mean unknown |
| **`Titel`** | **1,467** | **72.7 %** | **occupation/rank free text, 889 distinct values** |
| `Biografi` | 619 | 30.7 % | short prose; 87 rows begin `g.m.` (*gift med*, "married to") |
| `Relationer` | 31 | 1.5 % | genealogical microdata — `d.a.` (daughter of), `s. af` (son of), `f.` (née) |
| `MusID` | 5 non-zero | 0.2 % | drop |

Three of these are worth more than their fill rate suggests, because
`hca-open-repo` derives the same facets heuristically from prose:

- **`Titel`** is a *stated* occupation ("Forfatter", "Komponist",
  "Skuespillerinde", "Forlagsboghandler", "Biskop"). `hca-open-repo`'s
  9-term role facet (`data/normalized/person_role.csv`:
  `Forfatter/Digter`, `Musiker/Scenekunst`, `Embedsmand/Jura/Politik`, …)
  is *inferred* from description terms and reaches 5,564 of 10,228
  persons. `Titel` is independent, source-stated evidence for the same
  facet on 1,467 correspondents. It is dirty in exactly one predictable
  way — case is inconsistent (`Forfatter` 109 / `forfatter` 18;
  `Skuespiller` 16 / `skuespiller` 14) — so it needs case-folding before
  vocabulary mapping, not free-text storage pretending to be a code.
- **`Hjemland`** is a *stated* country, against `hca-open-repo`'s
  `person_ethnic_descriptors.csv`, which is parsed out of register prose.
  §3.3 shows the two disagree often enough to matter.
- **`Relationer`/`Biografi`** carry family relations
  ("g.m. Peter Jonas Collett", "barnebarn af Jonas Collin",
  "s. af regimentskirurg Joachim Otto Møller (1781-1873)") — a person↔person
  relation type that has **no counterpart anywhere in `hca-open-repo`**.
  This is genuinely new content, not a duplicate of something the
  register already has. It is also unparsed prose: model it as evidence
  (§5), not as a resolved edge.

### 2.4 What the relational form adds over `BrevBasen.csv`

| `correspondence-integration.md` observed (flat export) | The dump shows |
|---|---|
| 1,919 distinct persons | **2,018** — the export only reaches persons who appear in a letter; 99 registered correspondents have no surviving letter relation |
| `BrevID='NULL'`, a 1,848-row junk bucket to filter heuristically | Structural: `brev` (13,585) and `brev_person` (27,637) are separate tables; orphans are 4 rows against 2 unknown `PersonID`s, findable by join, not by string sniffing |
| Sender/recipient as two flattened join slots (`PersID`/`PersID19`) needing a cross-slot consistency check | One `brev_person` table, `Relation ENUM('afsender','modtager')`, clean: 13,922 / 13,715, no third value |
| Person↔letter counts derived by de-duplicating rows per `BrevID` | Direct: `brev_person` is already at relation grain |
| — | **Tables not in the export at all**, listed in §8 |

The export's own §7 warnings survive intact and still apply: partial
`-00` dates, embedded HTML in names and prose, one post-mortem letter
date. They are properties of the data, not of the export format.

---

## 3. The crosswalk: `brevperson` → `entities.csv`

### 3.1 Result

Method as in `correspondence-integration.md` §2 — diacritic-stripped
lowercase surname lookup, confirmed by a shared 4-digit birth year parsed
from the `"Efternavn, Fornavn (byyy–dyyy)"` label convention — re-run
against the dump's clean `brevperson` table rather than the flattened
export, with embedded HTML stripped before normalization:

| Outcome | Persons | % of 2,018 |
|---|---|---|
| Surname + birth year → exactly one register candidate | **1,135** | 56.2 % |
| Surname + birth year → several candidates | 86 | 4.3 % |
| Surname matches, birth year missing or non-confirming | 403 | 20.0 % |
| No surname match in the register at all | 346 | 17.1 % |
| No surname on the `hca_db` side to key on | 48 | 2.4 % |

The 1,135 confident matches cover **9,677 of 14,527 (66.6 %)** of the
non-Andersen sides of letter relations — i.e. two thirds of the
correspondence volume is reachable through a single-candidate match,
concentrated as expected in the largest correspondences (Edvard Collin
742 relation rows, Henriette Wulff 597, Henriette Oline Collin 479,
Jonas Collin d.æ. 466, Dorothea Melchior 451, B. S. Ingemann 396).

Andersen himself (`PersID=1`, 13,110 relation rows, 47.4 % of the table)
does **not** fall out of this method — surname *Andersen* + birth year
1805 does not resolve to a single register candidate. He is the one row
that should be hard-coded from a verified identity rather than matched.

### 3.2 The 1,135 are candidates, not links — measured, not assumed

Death year is available on both sides for 1,130 of the 1,135 confident
matches and was **not** used as a matching key, which makes it a free
hold-out test. **32 of 1,130 (2.8 %) disagree.**

That is the number that decides the workflow. A 2.8 % demonstrable
conflict rate on the *strongest* tier means the weaker tiers are worse,
and it means these rows must not be auto-promoted into anything a build
script or a public page reads. Each conflict is one of two things — a
same-surname/same-birth-year namesake wrongly joined, or a real
date discrepancy between two independently curated registers — and
telling them apart is human work. This is CLAUDE.md's fact-check rule
("verify before writing; never substitute an unverified guess for a
confirmed value") arriving as a measured error rate rather than a
principle.

The schema therefore stores **the tier and the evidence, not a verdict**
(§4), and the single existing verified row —
`data/curated/breve_person_crosswalk.csv`, `Reg0042200` ↔ `PersID=151`
(Fredrika Bremer) — is the shape every promoted row must reach.

### 3.3 Nationality is a second, independent hold-out

Comparing `brevperson.Hjemland` against `person_ethnic_descriptors.csv`'s
`nationality_label` for the confident matches, over the 11 countries with
an unambiguous Danish adjectival form: **259 agree, 62 disagree**, 814
not comparable (no descriptor parsed on the register side). A ~19 %
disagreement rate on comparable rows is not primarily a matching error —
it is the expected gap between *citizenship/home country* (`Hjemland`)
and *ethnic descriptor as the register's prose describes the person*
("holstensk", "preussisk", and the umbrella overlaps already documented
in `hca-open-repo`'s `nation_umbrellas_da.csv`, where *holstensk* sits
under both *dansk* and *tysk*). Model it as two attributed claims from
two sources, never reconcile it into one column, and use disagreement as
a review signal rather than an error.

---

## 4. Schema delta

Changes to [`postgres-schema-design.md`](postgres-schema-design.md) §3
only. Depends on `entity`, `person`, `import_batch`,
`reconciliation_status` and `entity_alias` as defined there and in
[the parsed-works addendum](postgres-schema-design-addendum-parsed-works.md).

```sql
-- ============================================================
-- 4.1  External person registers — namespaced, never merged
-- ============================================================
-- hca_db contains two independent person registers whose ID spaces both
-- start at 1 and overlap numerically (§2.1). Storing a bare integer
-- would silently collide. register_key namespaces them, and the
-- composite PK makes a collision a constraint violation rather than a
-- wrong join.
CREATE TABLE external_person_register (
  register_key  TEXT PRIMARY KEY,   -- 'hca_db.brevperson' | 'hca_db.person'
  label         TEXT NOT NULL,
  source_system TEXT NOT NULL,      -- 'hca_db'
  url_template  TEXT,               -- brevperson: '…/brevbase/person.html?breve=sendt&pid={id}'
  notes         TEXT
);

-- One row per source person, stored VERBATIM. This is not an entity and
-- never becomes one: no row here creates or edits an `entity` row.
-- hca-open-repo's register is the basis; this is an authority mapped
-- against it.
CREATE TABLE external_person (
  register_key    TEXT NOT NULL REFERENCES external_person_register(register_key),
  external_id     TEXT NOT NULL,      -- brevperson.PersID (the PUBLIC key, §2.2)
  surrogate_id    TEXT,               -- brevperson.ID; kept for dump traceability only
  given_name_raw  TEXT,               -- Fornavn, HTML NOT stripped
  family_name_raw TEXT,               -- Efternavn, incl. the ", f. X" née convention
  title_raw       TEXT,               -- Titel — occupation/rank, 889 distinct values
  born_date_raw   TEXT,               -- '1803-00-00' kept as text: NOT a DATE (§5.1)
  died_date_raw   TEXT,
  born_year       SMALLINT,           -- derived, NULL when the raw year is 0000
  died_year       SMALLINT,
  home_country    TEXT,               -- Hjemland; 'ukendt'/'' normalized to NULL
  biography_raw   TEXT,               -- Biografi
  relations_raw   TEXT,               -- Relationer
  source_updated_at TIMESTAMPTZ,      -- brevperson.Timestamp
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  PRIMARY KEY (register_key, external_id)
);
CREATE INDEX external_person_family_idx ON external_person (lower(family_name_raw));
CREATE INDEX external_person_born_idx   ON external_person (born_year) WHERE born_year IS NOT NULL;

-- ============================================================
-- 4.2  The mapping table  ← the point of this addendum
-- ============================================================
-- Deliberately NOT a nullable external_person_id column on `person`:
--   * a mapping is a claim with evidence, a tier and a reviewer, not an
--     attribute of the person;
--   * unmatched and rejected candidates must survive (§3.2's 32
--     conflicts are the most valuable rows in the table, not waste);
--   * one register person can attract several candidate mappings before
--     review, and the schema must be able to hold them side by side.
CREATE TYPE person_match_tier AS ENUM (
  'surname_birthyear_unique',   -- 1,135 — the confident tier
  'surname_birthyear_multi',    --    86
  'surname_only',               --   403
  'no_surname_match',           --   346
  'no_surname_key',             --    48
  'manual'                      -- proposed by a human, not by the matcher
);

CREATE TABLE person_external_map (
  map_id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id       BIGINT NOT NULL REFERENCES person(entity_id),
  register_key    TEXT NOT NULL,
  external_id     TEXT NOT NULL,
  status          reconciliation_status NOT NULL DEFAULT 'candidate',
  match_tier      person_match_tier NOT NULL,
  -- Hold-out evidence: computed at match time, NEVER used as a matching
  -- key, so it stays an independent test (§3.2, §3.3).
  died_year_agrees      BOOLEAN,   -- NULL = not comparable
  nationality_agrees    BOOLEAN,
  letter_count          INTEGER,   -- from brev_person; the review-priority key
  -- Promotion record — mirrors data/curated/breve_person_crosswalk.csv
  verified_via    TEXT,
  verified_by     TEXT,
  verified_at     TIMESTAMPTZ,
  notes           TEXT,
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  FOREIGN KEY (register_key, external_id)
    REFERENCES external_person(register_key, external_id),
  UNIQUE (entity_id, register_key, external_id)
);
CREATE INDEX pem_entity_idx   ON person_external_map (entity_id);
CREATE INDEX pem_external_idx ON person_external_map (register_key, external_id);
-- Review queue: biggest correspondences first, exactly the ordering
-- correspondence-integration.md §9 Phase 1 calls for.
CREATE INDEX pem_review_queue_idx ON person_external_map (letter_count DESC)
  WHERE status = 'candidate';

-- A confirmed mapping is 1:1 IN BOTH DIRECTIONS. Two partial unique
-- indexes, not a table-level UNIQUE, so candidates and rejections stay
-- unconstrained — the same pattern authority_identifier and
-- place_geocode already use in §3 of the main document.
CREATE UNIQUE INDEX pem_one_confirmed_per_entity
  ON person_external_map (entity_id, register_key) WHERE status = 'confirmed';
CREATE UNIQUE INDEX pem_one_confirmed_per_external
  ON person_external_map (register_key, external_id) WHERE status = 'confirmed';

-- A conflicting hold-out must never be promoted silently. This is §3.2's
-- 2.8 % expressed as a constraint rather than a warning in prose.
ALTER TABLE person_external_map ADD CONSTRAINT pem_conflict_needs_a_reason
  CHECK (status <> 'confirmed' OR died_year_agrees IS NOT FALSE OR verified_via IS NOT NULL);

-- ============================================================
-- 4.3  Source-stated attributes as attributed claims, not overwrites
-- ============================================================
-- hca-open-repo is the basis: its derived facets stay authoritative and
-- untouched. hca_db's stated Titel/Hjemland land here as a second
-- opinion, joined through the crosswalk, and only ever promoted into a
-- register facet by an editor.
CREATE TABLE external_person_claim (
  claim_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  register_key  TEXT NOT NULL,
  external_id   TEXT NOT NULL,
  claim_kind    TEXT NOT NULL
    CHECK (claim_kind IN ('occupation', 'home_country', 'family_relation', 'biography')),
  claim_text    TEXT NOT NULL,          -- verbatim source string
  claim_text_folded TEXT,               -- case-folded 'Titel' for vocabulary mapping (§2.3)
  mapped_role_key TEXT REFERENCES person_role(role_key),  -- NULL until mapped
  FOREIGN KEY (register_key, external_id)
    REFERENCES external_person(register_key, external_id)
);
CREATE INDEX epc_external_idx ON external_person_claim (register_key, external_id);
CREATE INDEX epc_kind_idx     ON external_person_claim (claim_kind);
```

### 4.4 What does **not** need a new table

- **The ", f. X" née convention.** `brevperson.Efternavn` values such as
  `"Collin, f. Thyberg"` and `"Serre, f. Hammerdörfer"` are the same
  convention `entities.csv` uses in 1,504 of its 10,228 person labels
  (`"Aarestrup, Caroline, f. Aagaard (1808–1897)"`). Split on import and
  emit an `entity_alias` row with `alias_kind='variant_spelling'`; the
  parsed-works addendum's §2.5 table already covers it. **This is also
  the highest-value unexploited matching signal**: a maiden name gives
  the matcher a second surname to key on, which is exactly what the 403
  `surname_only` rows lack.
- **Letter counts.** Derived from `brev_person`; cached on
  `person_external_map.letter_count` purely as a review-queue sort key,
  never as the source of truth.
- **External URLs.** `external_person_register.url_template` +
  `external_id` builds the `pid=` link. The `breve=sendt` segment stays
  what `correspondence-integration.md` §4 established: a literal that is
  verified to work, not a parameter whose value set is known.

---

## 5. The letter layer

The crosswalk is only worth building because something hangs off it.
Minimal shape, at the grain the dump actually has:

```sql
CREATE TYPE letter_relation AS ENUM ('afsender', 'modtager');   -- sender | recipient

CREATE TABLE letter (
  letter_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  legacy_brev_id  TEXT NOT NULL UNIQUE,   -- brev.ID; the bid= in the public URL
  written_date_raw TEXT,                  -- '1844-07-00' — text, not DATE (§5.1)
  written_date    DATE,                   -- NULL unless the raw date is complete
  date_precision  TEXT CHECK (date_precision IN ('day','month','year')),
  location_raw    TEXT,                   -- Lokation — 'Ukendt' on the large majority
  provenance_no   TEXT,                   -- HerkomstNr, e.g. 'HCA 1971/356-0001'
  import_batch_id BIGINT REFERENCES import_batch(batch_id)
);

CREATE TABLE letter_participant (
  letter_id     BIGINT NOT NULL REFERENCES letter(letter_id),
  register_key  TEXT NOT NULL,
  external_id   TEXT NOT NULL,
  relation      letter_relation NOT NULL,
  PRIMARY KEY (letter_id, register_key, external_id, relation),
  FOREIGN KEY (register_key, external_id)
    REFERENCES external_person(register_key, external_id)
);
```

Note what `letter_participant` references: **`external_person`, not
`person`.** A letter's participants are facts in `hca_db` and stay
resolvable whether or not the person was ever crosswalked — a register
link is an *additional* join through `person_external_map`, not a
precondition for loading the correspondence. Only 56.2 % of correspondents
have even a candidate match; wiring letters to `person` directly would
make the other 43.8 % unloadable.

**`MetaTekst` stays the highest-value first target** — the 389
hand-written "Dagbog …" diary citations identified in
`correspondence-integration.md` §5 are a human-curated letter↔diary
crosswalk requiring no date heuristics. In this schema they resolve to
`letter` ↔ `diary_line`/`diary_page` links and should be modelled as
their own table when that work starts; they are also the validation set
for any automated date-join built later.

---

## 6. Data-quality issues specific to this source

1. **`0000-00-00` is not a date and MySQL's zero-date has no PostgreSQL
   equivalent.** 325 of 2,018 birth dates and 343 of 2,018 death dates
   are the zero value; partial dates (`1803-00-00`, `1842-07-00`) are
   common. Import raw-as-text plus a derived year, exactly as §4.1 and §5
   do — a naive `date` cast fails outright or silently corrupts. This is
   the same convention `hca-open-repo`'s `short_date()`/`formatDate` helpers
   already handle, so no new date logic is needed on the presentation side.
2. **Embedded HTML in name fields.** `PersID=1`'s `Fornavn` is
   `<acronym title="Hans Christian">H.C.</acronym>`; `Biografi` and
   `Relationer` carry `<a href>`, `<p>` and HTML entities (`&quot;`,
   `&#269;`). Store raw, strip for matching and display — never strip on
   import.
3. **`Titel` case inconsistency** (§2.3) — fold before vocabulary
   mapping.
4. **`ENGINE=MyISAM`, no foreign keys, `latin1` table charset under a
   `SET NAMES utf8mb4` dump.** No referential guarantee is inherited;
   the 4 orphan `brev_person` rows (§2.2) are what that costs. Verify the
   encoding of the actual byte stream on import rather than trusting
   either declaration.
5. **`Hjemland` is a 93-value ENUM including composites**
   (`Danmark-Frankrig`, `Sverige-Norge`, `Tyskland-Ungarn`) and
   historical polities (`Preussen`, `Holsten`, `Bøhmen`, `Dansk
   Vestindiske Øer`). Do not map it onto a modern country code list; it
   is closer in kind to `hca-open-repo`'s `nation_umbrellas_da.csv` than
   to ISO 3166.
6. **The dump is a 2023-06-29 snapshot**, three years stale as of this
   writing. `source_updated_at` (from `brevperson.Timestamp`) and
   `import_batch` together are what make a later re-import diffable
   rather than a full reload.

---

## 7. How the numbers here were produced

A tokenizing reader over `hca_db.sql` that walks `CREATE TABLE` blocks
and parses the `INSERT` tuples, handling backslash escapes and embedded
quotes/commas (a naive `split(',')` mis-parses `Biografi` prose badly).
Row counts are tuples per table block. Crosswalk matching: `Efternavn`
truncated at the first comma (to drop the ", f. X" tail), HTML tags
removed, NFKD-normalized, combining marks and non-letters stripped,
lowercased; register-side surnames parsed from `entities.csv` labels with
`^(?P<sur>[^,(]+),\s*(?P<given>[^(]*)\((?P<b>\d{4})?\s*[–-]?\s*(?P<d>\d{4})?\)`;
confirmation on a shared 4-digit birth year. Death-year and nationality
hold-outs computed after matching, never as keys. Letter counts from
`brev_person` at relation grain (27,637 rows), not de-duplicated by
letter — stated as relation rows throughout, not as "letters", because
those are different numbers.

The 56.2 % confident rate here is higher than
`correspondence-integration.md` §2's 52.4 % from the flat export. The
difference is **not** a better matcher: it is a larger and cleaner
denominator (2,018 register persons vs 1,919 letter-bearing persons) plus
HTML stripping before normalization. Both figures are correct for their
own input; neither supersedes the other's method.

---

## 7b. Verification — the DDL in §4 and §5 was executed, not just written

Every `CREATE`/`ALTER` statement in this document was extracted from the
fenced blocks and run against a live **PostgreSQL 16.13** server, on top
of `postgres-schema-design.md` §3 and the parsed-works addendum §3, with
`ON_ERROR_STOP=1`. The full stack applies cleanly, in that order, from
an empty database. (The MySQL `brevperson` DDL quoted in §2.2 is source
material and is excluded from that run.)

The constraints were then behaviour-tested, not just compiled:

| Test | Expected | Result |
|---|---|---|
| `external_id='151'` in both registers at once | both persist — namespacing works | 2 rows ✓ |
| Second `confirmed` mapping for one register person | rejected | `pem_one_confirmed_per_entity` ✓ |
| One external person `confirmed` onto a second register person | rejected | `pem_one_confirmed_per_external` ✓ |
| Two *candidates* for the same register person | allowed | 2 rows ✓ |
| Promote a row with `died_year_agrees = false`, no `verified_via` | rejected | `pem_conflict_needs_a_reason` ✓ |
| Same promotion *with* `verified_via` | allowed | ✓ |
| `letter_participant` naming an unknown external person | rejected | composite FK ✓ |
| `letter_relation` value outside `afsender`/`modtager` | rejected | enum ✓ |

The fifth row is the one that matters: §3.2's 2.8 % death-year conflict
rate is enforced by the database, so a conflicting candidate cannot reach
`confirmed` without a human leaving a reason behind.

---

## 8. What this addendum does not cover

- **The works axis.** `vaerktitler` (3,916 rows) and `bibliografi`
  (16,301) are Index B and Index A of
  `hca_db_export/docs/authority-model.md`'s three-authority model, and
  they face `hca-open-repo`'s 3,708-row VÆRK-REGISTER — a crosswalk of
  the same shape as this one, at similar scale, with its own hold-out
  problems. It needs its own addendum. Note that
  `hca_db_export`'s authority model has **no person axis at all**: its
  Index A/B/C are all work-shaped. The person crosswalk designed here is
  a fourth axis alongside it, not an instance of it.
- **Tables in the dump with no `hca-open-repo` counterpart yet**, listed
  so they are not rediscovered as surprises: `sted` (1 row — a place
  table that exists structurally but is empty in practice, which closes
  `correspondence-integration.md` §6's open question negatively:
  **the dump does not supply a place register**), `motiv` (123) and
  `motiv_tekststed` (1,847) — a motif index over text locations,
  `tekststed` (693), `almanak` (5,163), `dagbog` (352),
  `tidstavle_data` (1,446) — a chronology, and `brevnote` (7,686).
  Several are directly relevant to the diary side of `hca-open-repo` and
  deserve their own assessment.
- **Places and artworks in the correspondence.** Unchanged from
  `correspondence-integration.md` §6: still a new indexing task, and the
  dump does not change that verdict — it confirms it, since `sted` turns
  out to be empty.

---

## 9. Revised phasing

Supersedes `correspondence-integration.md` §9 for the person axis; its
Phase 0 (`MetaTekst`) is unchanged and still the best first move.

| Phase | Work | Gate |
|---|---|---|
| **0** | `MetaTekst` diary↔letter extraction (unchanged) | 389 hand-curated pairs, no matching risk |
| **1** | Load `external_person`, `letter`, `letter_participant` from the dump verbatim (§4.1, §5) | No crosswalk needed; loads 100 % of the source |
| **2** | Populate `person_external_map` at all five tiers with hold-out flags computed (§4.2) | Zero rows promoted to `confirmed` in this phase |
| **3** | Add maiden names to `entity_alias` and re-match the 403 `surname_only` rows on both surnames (§4.4) | Re-run; measure the tier shift before and after |
| **4** | Human review, `letter_count DESC`: Collin, Wulff, Melchior, Ingemann first; the 32 death-year conflicts as a separate queue | Promotion to `confirmed` requires `verified_via`, per the Bremer row's precedent |
| **5** | Surface `Brevveksling` links on person pages from confirmed rows only | Per `correspondence-integration.md` §4 and `hca-open-repo`'s `docs/external-links.md` |
| **6** | `Titel`/`Hjemland` claims into the role and nationality facets, editor-gated (§4.3) | Never auto-promoted into a register facet |

---

## 10. Open questions

1. **Does `person_external_map` belong in this repo's database, or does
   the verified subset stay a curated CSV in `hca-open-repo`?** Today
   `data/curated/breve_person_crosswalk.csv` holds exactly one verified
   row and that repo's CSVs are still the source of truth (per this
   repo's README). The honest interim answer is *both*: the CSV stays the
   editorial artefact, this table is what it loads into. That should be
   decided explicitly before either side grows a second row, not after.
2. **What promotes a row to `confirmed`?** The Bremer precedent used
   three independent confirmations (exact birth+death dates, the public
   `pid=`, and an independent diary-date corroboration). Is that the bar
   for all 1,135, or is it the bar only for high-volume correspondents?
   At three confirmations per row this is months of work; at one it is
   days and 2.8 % wrong.
3. **Is `hca_db.person` (§2.1) ever crosswalked?** The 12 overlapping
   historical figures argue for a `register_key='hca_db.person'` row in
   the same tables rather than a separate mechanism — the schema already
   allows it. But the other 664 are living scholars, which raises a data
   protection question this document is not the place to answer.
4. **Re-import cadence.** The dump is a 2023 snapshot of a live system.
   Is `hca_db` still being edited, and if so, does this repo re-import,
   or does the crosswalk freeze against this snapshot?
