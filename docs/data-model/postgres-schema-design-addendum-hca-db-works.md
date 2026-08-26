# Addendum: `hca_db` works, BFN, and the WEMI-aware crosswalk to the VÆRK-REGISTER

Status: proposal. Companion to
[`postgres-schema-design.md`](postgres-schema-design.md) — **read that
first** — and the sibling of
[the person addendum](postgres-schema-design-addendum-hca-db-persons.md),
whose §8 deferred exactly this work.

> **Governing principle, unchanged:** `hca-open-repo` is the **basis**,
> for structure and for content. `hca_db`'s two work indexes are
> **external authorities mapped against** it, never a second master. No
> `hca_db` row creates an `entity` row.

> **Second governing rule, specific to this axis:**
> `hca-open-repo`'s [`wemi-and-relations.md`](https://github.com/ogiermontanus/hca-open-repo/blob/main/docs/data-model/wemi-and-relations.md)
> defines the project's *simplified* WEMI: **any version with a new named
> creator (translator, adapter, arranger) is a new, independent Work;
> Expression is reserved for revised versions by the same creator.** That
> is deliberately stricter than canonical FRBR, and it is the rule this
> document applies — not FRBR's. §3 shows where `hca_db`'s structures
> land under it, including the 1,699 rows where the rule provably cannot
> fire.

All figures were counted from `hca_db_export/hca_db.sql` (phpMyAdmin dump,
MySQL 5.7, 2023-06-29), `hca-open-repo`'s `data/normalized/`, and — for
§2.3 — `ogierMontanus/sv-datarens` at commit `848532e`; §7 gives the
method.

---

## 0. A correction to the source specification, before anything else

`hca_db_export/docs/authority-model.md` and `docs/mapping-rules.md` both
name Index A's compiler **"Bjørn Frank Nielsen"**. That is wrong. The
bibliographer is **Birger Frank Nielsen**.

Three independent confirmations, per CLAUDE.md's verify-before-writing
rule:

1. **The dump's own editorial prose** — 78 occurrences of "Birger Frank
   Nielsen" inside `bibliografi.BibTekst`, zero of "Bjørn".
2. **The H.C. Andersen Centre's own bibliography site**, which publishes
   BFN: [andersen.sdu.dk/forskning/bib/bfn/](https://andersen.sdu.dk/forskning/bib/bfn/).
3. **Library/booktrade catalogue records** for *H.C. Andersen
   Bibliografi. Digterens danske Værker 1822-1875* (Hagerup, Copenhagen
   1942, 462 pp., foreword by H. Topsøe-Jensen).

The dump corroborates this structurally too: `bibliografi.BibKildeNavn`'s
ENUM contains the literal string `'Digterens danske Værker 1822-1875'` —
BFN's exact subtitle — paired with `BibKilde='bfn'`. **The spec's
"Bjørn" should be corrected in `hca_db_export`**; it is not a variant
spelling but a different given name, and it will mislead anyone searching
a catalogue.

---

## 1. What `hca_db` actually holds on the works axis

### 1.1 Index A and Index B are both present, and Index A is seven bibliographies, not one

| Table | Rows | Role |
|---|---|---|
| `bibliografi` | **16,301** | **Index A** — publications. But see below: this is *seven* bibliographies in one table. |
| `vaerktitler` | 3,916 rows / **1,508 distinct `VID`** | **Index B** — Værkfortegnelsen, the intellectual works. The row count is titles, not works (§1.3). |
| `bibliografi_rel` | **5,013** | The **stated, typed** publication↔work edges (§1.4). |
| `vaerktitler_genre` | 2,986 | **Not** a genre assignment table (§1.2). |
| `vaerktitler_bibliografi` | 28 | The title-group authority list — translators (§1.3). |
| `bibliografi_genre` | 5,139 | Bibliographers' own genre labels, Danish + English — covering only 31.5 % of `bibliografi`. |
| `bibliografi_person` | 209 | Publication↔person, `RelationType ENUM('forfatter','redaktør','tilegnet','udgiver','oversætter')`. |

`bibliografi.BibKilde` is an ENUM of **seven distinct source
bibliographies**, and BFN is a minority of the table:

| `BibKilde` | `BibKildeNavn` | Rows |
|---|---|---|
| `hcac` | H.C. Andersen-Centrets bibliografiske optegnelser | 9,232 |
| `aaj1` | H.C. Andersen-litteraturen 1875-1968 | 2,333 |
| `aaj2` | H.C. Andersen-litteraturen 1969-1994 | 1,802 |
| **`bfn`** | **Digterens danske Værker 1822-1875** | **1,274** |
| `aaj4` | H.C. Andersen-litteraturen 1999-2006 | 1,112 |
| `imc` | Den gyldne trekant | 288 |
| `aaj3` | H.C. Andersen-litteraturen 1995-1998 | 260 |

This distinction is load-bearing and the source spec does not make it:
**`bfn` records Andersen's own publications; the `aaj*` and `hcac`
records are secondary literature *about* Andersen.** They are different
kinds of object sharing one table, exactly as `person` and `brevperson`
were on the person axis. Only `bfn` is Index A in the authority model's
sense.

**BFN identifiers are already a normalized column, not free text.**
`BibKildeID varchar(6)` holds the BFN number with `BibKildeID_as_int` as
its sort key. Of the 1,274 BFN rows: **1,270 are bare integers**, 1 has a
letter suffix, 3 are blank; 1,272 distinct values, no duplicates among
the non-blank ones. `mapping-rules.md` §"Identifier Format" anticipates
messy inline forms (`BFN 98`, `BFN 109B`, "multiple spaces, parentheses,
commas, line breaks") and prescribes Stage 3 "extract every explicit BFN
reference … construct a normalized BFN index". **On this source that
stage is already done.** The extraction problem it describes belongs to
Index C (the TEI/XML and Laravel sources), which is not in this dump.

### 1.2 `vaerktitler_genre` is a VID block-allocation table — the trap in this schema

It looks like a per-work genre assignment: 2,986 rows, one per `VID`,
`Genre` ENUM. It is not. Every genre owns a **contiguous, pre-allocated
block of VID numbers**, and half the blocks are empty:

| Genre | VID block | Reserved | Used (has a title row) | Fill |
|---|---|---|---|---|
| Eventyr | 1–299 | 299 | 212 | 70.9 % |
| Eventyrsamling | 301–399 | 99 | 38 | 38.4 % |
| Roman | 401–499 | 99 | 6 | 6.1 % |
| Anden prosa | 501–599 | 99 | 6 | 6.1 % |
| Drama | 601–699 | 99 | 51 | 51.5 % |
| Rejseskildring | 701–799 | 99 | 25 | 25.3 % |
| Selvbiografi | 801–899 | 99 | 4 | 4.0 % |
| Biografi | 901–999 | 99 | 6 | 6.1 % |
| Samlede og blandede skrifter | 1001–1099 | 99 | 36 | 36.4 % |
| Afhandlinger, artikler, breve etc. | 1101–1199 | 99 | 42 | 42.4 % |
| Satire og humor | 1201–1299 | 99 | 7 | 7.1 % |
| Digtsamling | 1301–1399 | 99 | 8 | 8.1 % |
| Digtkredse | 1401–1499 | 99 | 6 | 6.1 % |
| **Digte** | **1501–2900** | **1400** | **1024** | 73.1 % |
| Enkelttryk | 2901–2999 | 99 | 37 | 37.4 % |
| **TOTAL** | | **2,986** | **1,508** | **50.5 %** |

So: **genre is a pure function of the VID**, every VID with a title row
has exactly one genre (0 title-only VIDs), and **1,478 of the 2,986 rows
describe reserved-but-unused numbers with no work behind them**. An
importer that treats this table as "2,986 works with genres" invents
1,478 phantom works. Import it as an *allocation map*
(`genre_vid_block`), and derive a work's genre from its VID.

The block boundaries also encode editorial intent worth preserving:
Andersen's own numbering gave the fairy tales the first 299 slots and the
poems 1,400.

### 1.3 `vaerktitler` is a title-variant table, and `TitelgruppeID` is a translator authority

3,916 rows over 1,508 works. `TitelgruppeID` joins to
`vaerktitler_bibliografi`, which turns out to be a list of **translation
authorities** — 24 groups, 11 of them naming a specific translator:

| Group | Authority | Lang | Title rows |
|---|---|---|---|
| **15** | **H.C. Andersen** | dan | **1,508** — exactly one per work: the canonical original |
| 3 | Jean Hersholt | eng | 203 |
| 4 | *(unnamed)* | fre | 175 |
| 16 | *(unnamed)* | swe | 162 |
| 14 | *(unnamed)* | nor | 160 |
| 2 | Perlet | ger | 157 |
| 23 | *(unnamed)* | fin | 157 |
| 1 | Dal & Dal 1992 | dan | 156 |
| 6 | *(unnamed)* | spa | 156 |
| 7 | *(unnamed)* | dut | 156 |
| 22 | *(unnamed)* | rus | 154 |
| 8 | *(unnamed)* | pol | 152 |
| 5 | Manghi & Rinaldi | ita | 151 |
| 10 | *(unnamed)* | por | 128 |
| 11 | *(unnamed)* | epo | 108 |
| 13 | *(unnamed)* | ice | 104 |
| 9 | *(unnamed)* | cze | 44 |
| 12 | *(unnamed)* | ina | 43 |
| 24 | Paula Hostrup-Jessen | eng | 28 |
| 18 | Bech | ita | 6 |
| 19 | C.B. Lohmeyer | eng | 2 |
| 17 | Mörch-Fasting | ger | 1 |
| 20 | Anne S. Bushby | eng | 1 |
| 21 | Horace Scudder | eng | 1 |
| **0** | *(no group row at all)* | — | 3 |

Group 15 having exactly 1,508 rows — one per distinct VID, no more, no
fewer — is what confirms the reading: it is the Andersen-original title
slot, and every other group is a translation of it.

`TID char(3)` is blank on 1,405 rows and otherwise carries small
integers; it does not behave as a stable key and should be preserved
verbatim without being given a meaning.

### 1.4 `bibliografi_rel` — the BFN→Work crosswalk already exists, hand-typed

5,013 edges from a publication to a work (`BibliografiID` → `VID`),
each carrying a **stated relation type** from an ENUM:

| `Relation` | Edges | All edges | From BFN rows only |
|---|---|---|---|
| `førsteudgivelse af` (first publication of) | 1,510 | ✓ | 1,339 |
| `del af` (part of) | 1,369 | ✓ | 1,172 |
| `JdMs nudanske digtudgave af` (de Mylius's modern-Danish poem edition of) | 917 | ✓ | 0 |
| `anbefalet udgave af` (recommended edition of) | 904 | ✓ | 0 |
| `oversættelse af` (translation of) | 206 | ✓ | 2 |
| `Anmeldelse af` (review of) | 56 | ✓ | 0 |
| `førsteudgave af` (first edition of) | 32 | ✓ | 28 |
| `udgave af` (edition of) | 19 | ✓ | 16 |
| `gendigtning af` (retelling of) | **0** | declared in the ENUM, never used | 0 |

**1,264 of the 1,274 BFN publications (99.2 %) carry at least one typed
work edge**, reaching 1,376 distinct VIDs. `mapping-rules.md`'s Stages
3–5 — extract BFN references, resolve them against Index A, build
BFN→Publication and Publication→Work mapping tables — are, for this
source, **already built and curated by hand**. The job here is to import
them without loss, not to reconstruct them.

The cardinality confirms the spec's superkey warning empirically rather
than by assertion: **max 69 works per BFN publication**, **max 178 BFN
publications per work**, and 124 works carry more than one. Any
one-to-one assumption breaks immediately.

### 1.5 Satellite tables

`vaerktitler_url` (791 — full text at ADL and elsewhere, with a
`URLRating`), `vaerktitler_manuskripturl` (55 — manuscript facsimiles at
Odense Bys Museer), `vaerktitler_dslkommentar` (172 — DSL commentary,
with page numbers), `kbeventyrkode` (172 — a Royal Library fairy-tale
code, `ek2` style, plus an integer form), `bibliografi_bibtekst_e` (31 —
English annotations, a very thin overlay on 16,301 rows).

`kbeventyrkode` is the only external authority identifier on this axis
and belongs in the existing `authority_identifier` table
(`system_key='kb_eventyrkode'`), not in a new one.

---

## 2. The crosswalk target: what `hca-open-repo`'s VÆRK-REGISTER actually is

### 2.1 Scope — three quarters of the register has no possible counterpart

The register's 3,708 `work` entities split by wing:

| `genre_h2` | Works | Exact normalized-title hit anywhere in `hca_db` |
|---|---|---|
| ANDRE FORFATTERE | 1,545 | 5 (**0.3 %**) |
| BILLEDKUNST | 941 | 0 (**0.0 %**) |
| MUSIK | 454 | 1 (**0.2 %**) |
| **H. C. ANDERSEN** | **768** | **220 (28.6 %)** |

`hca_db`'s work tables are **Andersen-only**. The other three wings are
works by other people that Andersen saw, read or heard — Murillo
paintings, Dickens novels, operas — and they are structurally absent from
this source. **The works crosswalk is scoped to the 768-row H. C.
ANDERSEN wing: 20.7 % of the register.** The remaining 2,940 works need a
different authority entirely (Wikidata is already doing this job for the
BILLEDKUNST wing via `works_wikidata.csv`).

This mirrors the person axis exactly: one register in the dump is in
scope, the rest is a different kind of object.

### 2.2 The register wing is WEMI-mixed — and that, not title noise, is why matching caps out

Matching the 768 HCA-wing labels (year-parentheses and `Se ogsaa` tails
stripped, diacritics folded, `aa`→`å` normalized) against `vaerktitler`:

| `form_h3` | Matched | Total | Rate |
|---|---|---|---|
| Eventyr | 136 | 286 | **47.6 %** |
| Rejseskildringer | 9 | 21 | 42.9 % |
| Skuespil og Operatekster | 23 | 56 | 41.1 % |
| Romaner og Noveller | 9 | 33 | 27.3 % |
| Afhandlinger, Artikler m.m. | 4 | 19 | 21.1 % |
| Selvbiografier | 1 | 10 | 10.0 % |
| **Digte** | **13** | **338** | **3.8 %** |
| Samlede og blandede Skrifter | 0 | 5 | 0.0 % |
| **All** | **195** | **768** | 25.4 % |

The Digte row is not a matching failure. It is the two registers sitting
at **different WEMI levels for the same material**:

- `hca_db` models poems at **Work** level — 1,024 individual poems, each
  titled with its incipit: `Det døende barn (Moder, jeg er træt, nu vil
  jeg sove)`, `Hjertesuk til månen (Hulde måne, stig fra skyen frem)`.
- The register's Digte wing is mostly at **Manifestation** level — the
  *collections* the poems were printed in: `Samlede Digte (1833)`,
  `Phantasier og Skizzer (1831)`, `Gedichte. Deutsch von H. Zeise (1846)`,
  `Kjendte og glemte Digte (1867)`.

A 3.8 % title match between a poem and the anthology containing it is the
*correct* result. The same pattern explains the 0 % on Samlede og
blandede Skrifter (`Samlede Skrifter I-XXXIII (1853-79)`, `Gesammelte
Werke (1847-72)` — pure Manifestations) and much of the Eventyr miss
(`Nye Eventyr. 2. Bind. 1. Samling (1847)`, `Historier. Med 55 Ill. af
V. Pedersen (1855)`).

**Consequence for the schema:** the register wing carries Work-level and
Manifestation-level entries side by side in one flat list with **no
column distinguishing them**, and the crosswalk must record which level
each mapping joins at, or it will silently equate a poem with the book it
appeared in. That is what `wemi_level` and `map_kind` in §4 are for.

**Consequence for the matcher:** the incipit is the join key that would
actually work for poems, and both sides already expose it —
`vaerktitler` inside the title parenthesis, the register inside
guillemets (`A. Henselt (»Fra Strengene flyver en Fuglehær -«)`), which
`wemi-and-relations.md` parsing rule 3 already names as a title marker.
**This has now been built and measured against a third source — see §2.3.**

### 2.3 The incipit crosswalk, built on `sv-datarens`

The join key §2.2 predicted is supplied, already normalized, by a fourth
repository: **`ogierMontanus/sv-datarens`** — the TEI/XML working copy of
*Samlede Værker* (SV), 29 files covering volumes 7–18. It was attached to
this session read-only and is the first Index-C-adjacent source the
project has had in hand (§6 still applies: this is the edition text, not
the Laravel relational layer).

**What SV marks up.** In the two poetry volumes, every poem's first line
carries an explicit incipit tag:

```xml
<l rend="firstIndent" subtype="firstline"
   corresp="Jeg drømte – dog en Drøm var det ei ganske!"
   xml:id="jeg-droemte-dog-en-droem-var-det-ei-ganske">Jeg drømte – dog en Drøm …</l>
```

Three usable fields, not one: `subtype="firstline"` identifies the line,
`@corresp` holds the **editors' own punctuation-normalized incipit**, and
`@xml:id` is a **slug of it** (`æ`→`ae`, `ø`→`oe`, lowercased,
hyphenated). The project does not have to invent a normalization — SV
ships one, made by the edition's own editors.

| SV volume | Tagged `firstline` elements |
|---|---|
| `Andersen 7 - Digte I` | 358 |
| `Andersen 8 - Digte II` | 474 |
| **Total** | **832** (830 carry both `@corresp` and `@xml:id`) |

Only the two poetry volumes carry the tagging; the other 27 files have
none. `@corresp` differs from the rendered line text on 69.7 % of rows —
it is the *stripped* form (trailing commas, semicolons and closing
guillemets removed), which is exactly what a join wants.

**Method.** Fold both sides to a comparison key: lowercase, `æ/ø/å` →
`ae/oe/aa`, unify the `ei`/`ej` orthography (SV keeps Andersen's original
spelling, `hca_db` modernises — `ei forsvunden` vs `ej forsvunden`),
strip non-alphanumerics. Exact key match first, then a 24-character
prefix match for truncated titles.

**Results, all three sides:**

| Link | Reached | Of |
|---|---|---|
| `hca_db` poem → SV firstline | **547** | 832 SV firstlines (65.7 %) |
| Register poem → SV firstline | **25** | 325 register poem entries |
| **Three-way** (register ↔ SV ↔ `hca_db`) | **17** | — |

For the register — the side that matters for this crosswalk:

| Method | Matched | of 325 real work entries |
|---|---|---|
| Title alone (§2.2's baseline) | 13 | 4.0 % |
| **SV incipit** | **25** | **7.7 %** |
| Overlap | 3 | — |
| **Combined** | **35** | **10.8 %** |

**22 register poems become reachable that no title matcher could find** —
a 2.7× improvement on the baseline. The reason it works is visible in the
pairs: the two registers frequently give the same poem completely
different titles, and only the incipit connects them.

| Register label | `hca_db` title | Joined via |
|---|---|---|
| `»Det nye Aar kommer susende -«` | `Ved årsskifter III` | `det-nye-aar-kommer-susende` |
| `Paa Nytaarsmorgen (»Du Evighedens Gaade -«)` | `Et besøg i Portugal 1866 6` | `du-evighedens-gaade` |
| `»Barn Jesus i en Krybbe laae -« (Aarets tolv Maaneder)` | `December` | `barn-jesus-i-en-krybbe-laae` |

No title normalization reaches those. The incipit does.

Two candidate files, following the project's propose→verify convention —
**neither is human-verified, and neither should be consumed by a build
script as-is**:

- [`exports/poem-incipit-crosswalk-candidates.csv`](exports/poem-incipit-crosswalk-candidates.csv)
  — 25 register↔SV rows, 17 of them three-way, sorted three-way first.
- [`exports/hca-db-poem-to-sv-firstline-candidates.csv`](exports/hca-db-poem-to-sv-firstline-candidates.csv)
  — 547 `hca_db`↔SV rows.

**Why the register side caps at 7.7 % — an undocumented register
convention.** 120 of the register's 325 poem entries carry a **leading
asterisk** (`*»Bag Rendsborgs gamle Volde -«`). That asterisk appears on
**120 of 3,708 works and nowhere else in the register** — only on
H. C. ANDERSEN-wing poems. It is a near-perfect predictor of absence from
SV:

| Register poem entries | With an incipit | Matched to SV |
|---|---|---|
| **Not** `*`-marked | 61 | **24 (39.3 %)** |
| `*`-marked | 111 | **4 (3.6 %)** |

Spot-checking confirms the pattern rather than just correlating with it:
`*»Aabne Strand ved Corselitze -«` and `*»Da Skotlands Skjalde deres
Sange skrev -«` (Til Miss Ross 1847) are occasional and album verse that
SV 7–8 do not print as poems at all — the phrases occur in SV only inside
commentary notes.

So the ceiling is set by **what the register contains**, not by the
matcher: on the non-asterisked entries the incipit method reaches 39.3 %,
which is an order of magnitude better than title matching. **What the
asterisk actually denotes editorially is not documented anywhere in
`hca-open-repo` and should be confirmed with the editors** — this
document establishes only that it predicts absence from SV, which is
enough to use it as a routing signal and not enough to state its meaning.
A further 13 poem entries are `se:` redirects (aliases, not works) and
are excluded from every denominator above.

**Remaining gap.** 285 of the 832 SV firstlines are reached from neither
register nor `hca_db`, and SV covers only volumes 7–8 for this tagging.
Volume 9 (*Blandinger*) has no `firstline` markup, so poems printed there
are unreachable by this method until it is tagged.

---

---

## 3. Mapping `hca_db` onto the project's simplified WEMI

### 3.1 Where each structure lands

| `hca_db` structure | Simplified-WEMI level | Note |
|---|---|---|
| `vaerktitler` `VID` | **Work** | the intellectual work |
| `vaerktitler` row, group 15 | Work's original title | one per VID |
| `vaerktitler` row, group ≠ 15, **named** authority | **new Work** + `translation_of` | the project rule fires (§3.2) |
| `vaerktitler` row, group ≠ 15, **unnamed** | **undecided** | the rule cannot fire (§3.2) |
| `bibliografi` row (`bfn`) | **Manifestation** | a publication |
| `bibliografi` row (`aaj*`, `hcac`) | *not in WEMI at all* | secondary literature about Andersen |
| `bibliografi_rel` edge | the W/E/M relation itself | typed by the source (§3.3) |
| `kbeventyrkode` | authority identifier | not a WEMI level |

### 3.2 The named-creator rule against the title groups

`wemi-and-relations.md`: *"Any version with a new named creator
(translator, adapter, arranger) is treated as a new, independent Work."*
Applied to §1.3's groups:

| Case | Groups | Title rows | Verdict |
|---|---|---|---|
| Group 15 — Andersen's own Danish title | 1 | 1,508 | the original Work |
| **Named** translator (Hersholt, Perlet, Dal & Dal, Manghi & Rinaldi, Hostrup-Jessen, Bech, Lohmeyer, Mörch-Fasting, Bushby, Scudder) | 10 | **706** | **rule fires → new Work + `translation_of` edge** |
| **Unnamed** translator (fre, swe, nor, fin, spa, dut, rus, pol, por, epo, ice, cze, ina) | 13 | **1,699** | **rule cannot fire — no named creator** |
| No group row at all | 1 (`TitelgruppeID=0`) | 3 | data error; quarantine |

This is the finding that most affects the design. **1,699 of the 2,408
non-canonical title rows — 70.6 % — are translations whose translator the
source does not name.** The project's rule keys on a *named* creator, so
these rows cannot be promoted to independent Works by rule, and they must
not be silently filed as Expressions either: Expression is reserved for
*same-creator* revision, and a translation into Finnish is definitionally
not that.

They are a **third state the current model has no slot for**: known to be
derivative, known not to be same-creator, creator unknown. The schema
must represent that honestly (`wemi_verdict='undetermined'` in §4) rather
than forcing a binary. Whether the project wants to extend the rule to
"any translation is a new Work regardless of attribution" is a genuine
editorial decision (§9.1), not something to infer — and it is worth
1,699 rows.

### 3.3 The relation ENUM against the project's `relation_type` vocabulary

`wemi-and-relations.md` derives relation types by matching text patterns
(`(Oversættelse)` → `translation_of`, `bearbejdet efter` →
`adaptation_of`). `hca_db` **states** them. The mapping:

| `bibliografi_rel.Relation` | Project `relation_type` | WEMI reading |
|---|---|---|
| `oversættelse af` | `translation_of` | new Work (named creator rule) |
| `gendigtning af` | `adaptation_of` | new Work — **but 0 rows use it** |
| `del af` | `part_of` | whole/part, same level |
| `førsteudgivelse af` | *(new)* `first_publication_of` | Manifestation → Work |
| `førsteudgave af` | *(new)* `first_edition_of` | Manifestation → Work |
| `udgave af` | *(new)* `edition_of` | Manifestation → Work |
| `anbefalet udgave af` | *(new)* `recommended_edition_of` | Manifestation → Work, **editorial recommendation** |
| `JdMs nudanske digtudgave af` | *(new)* `modernised_edition_of` | named editor (Johan de Mylius) → arguably a new Work under the rule (§9.2) |
| `Anmeldelse af` | **none — not a WEMI edge** | a review *about* a work; secondary literature |

Two of these need care rather than a straight import:

- **`Anmeldelse af` (56 rows) is not a derivation.** A review is a
  distinct publication *about* a work, not a version of it. Importing it
  into the same `relation` table as `translation_of` would corrupt
  exactly the multi-hop traversal that `wemi-and-relations.md` §"When does
  the type matter?" says typed edges exist for. Model it as an `about`
  edge in a separate namespace.
- **`anbefalet udgave af` (904 rows) is an editorial judgement**, not a
  bibliographic fact — it records which edition the Centre recommends. It
  is provenance-bearing opinion and should be flagged as such, not
  flattened into a neutral `edition_of`.

`wemi-and-relations.md`'s **"no shortcut edges"** rule applies directly:
where a BFN publication links to a work that is itself a translation of
another work, store both real edges and never synthesise the transitive
one.

---

## 4. Schema delta

Changes to [`postgres-schema-design.md`](postgres-schema-design.md) §3
only; depends on `entity`, `work`, `import_batch`,
`reconciliation_status`, `authority_identifier` and the person addendum's
`external_person_register` / `external_person`.

```sql
-- ============================================================
-- 4.1  External work registers — same namespacing pattern as persons
-- ============================================================
CREATE TABLE external_work_register (
  register_key  TEXT PRIMARY KEY,   -- 'hca_db.vaerktitler' | 'hca_db.bibliografi'
  label         TEXT NOT NULL,
  source_system TEXT NOT NULL,
  authority_index TEXT              -- 'A' (BFN/publications) | 'B' (Værkfortegnelsen)
    CHECK (authority_index IN ('A','B','C')),
  url_template  TEXT,
  notes         TEXT
);

-- Index B: the intellectual work. One row per VID (1,508), NOT per title.
CREATE TABLE external_work (
  register_key  TEXT NOT NULL REFERENCES external_work_register(register_key),
  external_id   TEXT NOT NULL,          -- vaerktitler.VID
  genre_source  TEXT,                   -- derived from the VID block (§1.2)
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  PRIMARY KEY (register_key, external_id)
);

-- §1.2: the VID block allocation, imported as what it is — an allocation
-- map — so the 1,478 reserved-but-unused numbers never become works.
CREATE TABLE genre_vid_block (
  genre_da      TEXT PRIMARY KEY,
  genre_en      TEXT,
  vid_low       INTEGER NOT NULL,
  vid_high      INTEGER NOT NULL,
  CHECK (vid_low <= vid_high),
  EXCLUDE USING gist (int4range(vid_low, vid_high, '[]') WITH &&)
);

-- §1.3: title-group authorities. is_named drives the WEMI verdict.
CREATE TABLE external_title_group (
  group_id      TEXT PRIMARY KEY,       -- vaerktitler.TitelgruppeID
  authority_name TEXT,                  -- NULL where the source names nobody
  lang          TEXT,                   -- SprogID
  is_named      BOOLEAN GENERATED ALWAYS AS
                  (authority_name IS NOT NULL AND btrim(authority_name) <> '') STORED
);

-- §3.2: the three-state verdict. 'undetermined' is a real answer, not a
-- gap to be filled in later by guessing.
CREATE TYPE wemi_verdict AS ENUM (
  'original',        -- group 15: Andersen's own Danish title
  'derived_work',    -- named creator -> new Work under the project rule
  'undetermined',    -- derivative, not same-creator, creator unnamed (1,699 rows)
  'quarantined'      -- TitelgruppeID with no group row (3 rows)
);

CREATE TABLE external_work_title (
  title_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  register_key  TEXT NOT NULL,
  external_id   TEXT NOT NULL,          -- the VID this title belongs to
  group_id      TEXT REFERENCES external_title_group(group_id),
  title_raw     TEXT NOT NULL,          -- verbatim, HTML entities intact
  incipit       TEXT,                   -- parsed from the title parenthesis (§2.2)
  tid_raw       TEXT,                   -- vaerktitler.TID — preserved, not interpreted
  wemi          wemi_verdict NOT NULL,
  FOREIGN KEY (register_key, external_id)
    REFERENCES external_work(register_key, external_id)
);
CREATE INDEX ewt_work_idx    ON external_work_title (register_key, external_id);
CREATE INDEX ewt_incipit_idx ON external_work_title (lower(incipit))
  WHERE incipit IS NOT NULL;
-- Exactly one original title per work.
CREATE UNIQUE INDEX ewt_one_original
  ON external_work_title (register_key, external_id) WHERE wemi = 'original';

-- Index A: publications. bib_source keeps BFN separate from the
-- secondary-literature bibliographies sharing the same table (§1.1).
CREATE TABLE external_publication (
  register_key  TEXT NOT NULL REFERENCES external_work_register(register_key),
  external_id   TEXT NOT NULL,          -- bibliografi.ID
  bib_source    TEXT NOT NULL,          -- 'bfn' | 'hcac' | 'aaj1'..'aaj4' | 'imc'
  is_andersen_publication BOOLEAN NOT NULL,   -- true only for 'bfn'
  bfn_number    TEXT,                   -- BibKildeID, verbatim ('109a' keeps its suffix)
  bfn_number_int INTEGER,               -- BibKildeID_as_int, the sort key
  title_raw     TEXT NOT NULL,
  published_date_raw TEXT,              -- '1847-00-00' as text (§6.1)
  published_year SMALLINT,
  lang          TEXT,
  media_type    TEXT,
  isbn          TEXT,
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  PRIMARY KEY (register_key, external_id)
);
CREATE INDEX epub_bfn_idx ON external_publication (bfn_number_int)
  WHERE bib_source = 'bfn';
-- BFN numbers are unique within BFN, but blank on 3 rows — partial index,
-- not a table constraint, so the blanks stay loadable.
CREATE UNIQUE INDEX epub_bfn_number_unique
  ON external_publication (bfn_number)
  WHERE bib_source = 'bfn' AND btrim(coalesce(bfn_number,'')) <> '';

-- ============================================================
-- 4.2  The stated, typed publication<->work edges (§1.4, §3.3)
-- ============================================================
-- work_relation_kind is the WEMI-bearing vocabulary; 'review_of' is
-- deliberately NOT in it (§3.3) and lives in its own table below, so a
-- graph traversal over derivations can never walk through a review.
CREATE TYPE work_relation_kind AS ENUM (
  'translation_of', 'adaptation_of', 'part_of',
  'first_publication_of', 'first_edition_of', 'edition_of',
  'recommended_edition_of', 'modernised_edition_of'
);

CREATE TABLE external_publication_work (
  rel_id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  register_key  TEXT NOT NULL,
  publication_id TEXT NOT NULL,
  work_register_key TEXT NOT NULL,
  work_external_id  TEXT NOT NULL,
  relation      work_relation_kind NOT NULL,
  relation_raw  TEXT NOT NULL,     -- the source ENUM string, always kept
  is_editorial_opinion BOOLEAN NOT NULL DEFAULT false,  -- 'anbefalet udgave af' (§3.3)
  FOREIGN KEY (register_key, publication_id)
    REFERENCES external_publication(register_key, external_id),
  FOREIGN KEY (work_register_key, work_external_id)
    REFERENCES external_work(register_key, external_id),
  UNIQUE (register_key, publication_id, work_register_key, work_external_id, relation)
);
CREATE INDEX epw_pub_idx  ON external_publication_work (register_key, publication_id);
CREATE INDEX epw_work_idx ON external_publication_work (work_register_key, work_external_id);

-- 'Anmeldelse af' (56 rows): a publication ABOUT a work. Separate table,
-- separate namespace — never a derivation edge.
CREATE TABLE external_publication_about_work (
  register_key  TEXT NOT NULL,
  publication_id TEXT NOT NULL,
  work_register_key TEXT NOT NULL,
  work_external_id  TEXT NOT NULL,
  about_kind    TEXT NOT NULL DEFAULT 'review',
  PRIMARY KEY (register_key, publication_id, work_register_key, work_external_id),
  FOREIGN KEY (register_key, publication_id)
    REFERENCES external_publication(register_key, external_id),
  FOREIGN KEY (work_register_key, work_external_id)
    REFERENCES external_work(register_key, external_id)
);

-- ============================================================
-- 4.3  The crosswalk  <- the point of this addendum
-- ============================================================
CREATE TYPE work_match_tier AS ENUM (
  'canonical_title_exact',   -- 193 — matched Andersen's own Danish title
  'variant_title_exact',     --   2 — matched a translated title
  'incipit',                 -- built in §2.3: 25 register + 547 hca_db candidates
  'title_fuzzy',
  'via_bfn_number',          -- strongest: an explicit BFN id on both sides
  'manual'
);

-- Which WEMI level the two sides meet at. Recording this is what stops a
-- poem being silently equated with the anthology that printed it (§2.2).
CREATE TYPE wemi_level AS ENUM ('work', 'expression', 'manifestation', 'mixed', 'unknown');

CREATE TABLE work_external_map (
  map_id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  entity_id     BIGINT NOT NULL REFERENCES work(entity_id),
  register_key  TEXT NOT NULL,
  external_id   TEXT NOT NULL,
  target_kind   TEXT NOT NULL CHECK (target_kind IN ('work','publication')),
  status        reconciliation_status NOT NULL DEFAULT 'candidate',
  match_tier    work_match_tier NOT NULL,
  -- WEMI bookkeeping (§2.2)
  register_level wemi_level NOT NULL DEFAULT 'unknown',  -- the hca-open-repo side
  source_level   wemi_level NOT NULL DEFAULT 'unknown',  -- the hca_db side
  level_agrees   BOOLEAN GENERATED ALWAYS AS (register_level = source_level) STORED,
  -- Hold-out evidence, never used as a matching key
  genre_agrees   BOOLEAN,
  year_agrees    BOOLEAN,
  verified_via  TEXT,
  verified_by   TEXT,
  verified_at   TIMESTAMPTZ,
  notes         TEXT,
  import_batch_id BIGINT REFERENCES import_batch(batch_id),
  UNIQUE (entity_id, register_key, external_id, target_kind)
);
CREATE INDEX wem_entity_idx   ON work_external_map (entity_id);
CREATE INDEX wem_external_idx ON work_external_map (register_key, external_id);
CREATE INDEX wem_review_idx   ON work_external_map (match_tier) WHERE status = 'candidate';

-- One confirmed mapping per side, per target kind — same partial-index
-- pattern as person_external_map.
CREATE UNIQUE INDEX wem_one_confirmed_per_entity
  ON work_external_map (entity_id, register_key, target_kind) WHERE status = 'confirmed';
CREATE UNIQUE INDEX wem_one_confirmed_per_external
  ON work_external_map (register_key, external_id, target_kind) WHERE status = 'confirmed';

-- A cross-level mapping is legitimate (a register Manifestation entry
-- genuinely maps to an hca_db Work) but must never be confirmed silently:
-- somebody has to say why the levels differ.
ALTER TABLE work_external_map ADD CONSTRAINT wem_cross_level_needs_a_reason
  CHECK (
    status <> 'confirmed'
    OR register_level = source_level
    OR verified_via IS NOT NULL
  );

-- A fuzzy title match is never self-evident: confirming one requires a
-- stated reason, the same bar §4.2 of the person addendum sets for a
-- conflicting hold-out.
ALTER TABLE work_external_map ADD CONSTRAINT wem_fuzzy_needs_a_reason
  CHECK (status <> 'confirmed' OR match_tier <> 'title_fuzzy' OR verified_via IS NOT NULL);
```

### 4.4 What does **not** need a new table

- **`kbeventyrkode`** → `authority_identifier` with
  `system_key='kb_eventyrkode'`; register it in `authority_system`.
- **`vaerktitler_url` / `_manuskripturl` / `_dslkommentar`** → the
  `work_source_citation` table the parsed-works addendum already
  introduced, plus `URLRating` as a confidence column. No new structure.
- **Translated titles as aliases.** Every `external_work_title` row with
  `wemi='undetermined'` should *also* surface as an `entity_alias`
  (`alias_kind='variant_spelling'`) on the mapped register work once
  confirmed — it makes 1,699 foreign-language titles searchable without
  committing to the Work/Expression question.
- **`bibliografi_person`** → the person addendum's
  `external_person_claim` pattern against `register_key='hca_db.person'`;
  its `RelationType` ENUM is the same shape as the parsed-works
  addendum's `work_agent.role`.

---

## 5. Data-quality issues specific to this axis

1. **`vaerktitler_genre` is an allocation map** (§1.2) — the single most
   likely import error on this axis. 1,478 of its 2,986 rows have no work.
2. **Zero and partial dates** — `bibliografi.UdgivDato` uses the same
   `0000-00-00` / `1847-00-00` convention as the person tables. Raw text
   plus a derived year, as in §4.1.
3. **HTML entities throughout titles** — `Dal &amp; Dal 1992`,
   `M&ouml;rch-Fasting`, `&#134;` in `bibliografi.Titel`. Unescape for
   matching and display; store raw.
4. **Orphans, since MyISAM enforces nothing**: `bibliografi_rel` carries
   2 `VID` values (`0`, `294`) with no work and 1 `BibliografiID` with no
   publication; `vaerktitler` has 3 rows whose `TitelgruppeID=0` has no
   group row. Small, but they must be quarantined rather than joined
   away silently.
5. **`BibTekst varchar(60000)`** is the annotation body and holds
   substantial HTML tables (this is where the 78 "Birger Frank Nielsen"
   attributions live). It is a document, not a field; do not try to
   normalize it.
6. **`bibliografi_bibtekst_e` covers 31 of 16,301 rows.** English
   coverage on this axis is effectively nil — do not design a bilingual
   UI around it.
7. **`gendigtning af` is declared but unused** (0 rows). Keep it in the
   vocabulary — its absence is a fact about this snapshot, not about the
   model — but do not report it as a supported relation.

---

## 6. What this addendum still does not cover

- **Index C is still only half present.** `sv-datarens` (§2.3) supplies
  the *edition text* side — TEI/XML for SV volumes 7–18 — and that is
  already enough to build the incipit crosswalk. What is still missing is
  the **Laravel relational layer**: anthology membership, parent/child
  work hierarchies, Laravel numeric IDs and UUIDs, which
  `mapping-rules.md` says to prefer over reconstructing relationships
  from XML. Until that arrives, Index C's *relationships* remain
  unaddressed. Note also that the `sv-datarens` files are a **working
  copy** (filenames carry editor names and dates —
  `_w_notes_Rebecca`, `_2024-06-23`), not a released edition snapshot; a
  citable version should be pinned before anything is promoted from it.
  The rest of the original caveat stands: Everything
  `architecture.md` and `mapping-rules.md` say about Index C — anthology
  membership from Laravel, embedded BFN references in XML, UUIDs, TEI
  ids — remains unaddressed and unverifiable from `hca_db_export` as it
  stands. The BFN extraction pipeline (Stages 3–4) that is redundant for
  this source is exactly what Index C will still need.
- **The 2,940 non-Andersen register works** (§2.1) need a different
  authority; Wikidata already serves the BILLEDKUNST wing.
- **The `aaj*`/`hcac` secondary literature** (14,739 rows) is a
  bibliography of Andersen scholarship. It is a legitimate and
  substantial resource, but it is not a work register and does not belong
  in this crosswalk.

---

## 7. How the numbers here were produced

The same tokenizing reader over `hca_db.sql` used for the person addendum
(`CREATE TABLE` block walk, `INSERT` tuple parse with backslash-escape and
embedded-quote handling). Title normalization: HTML entities unescaped,
tags stripped, NFKD, combining marks removed, lowercased, `aa`→`a`
folded, non-alphanumerics collapsed to single spaces. Register labels
additionally had year-parentheses (`(1847)`, `(1858-66)`) and
`- Se ogsaa …` tails removed before matching, since neither is part of
the title. VID-block boundaries computed as min/max per genre with a
contiguity check (all 15 blocks contiguous). "Used" VIDs are those with
at least one `vaerktitler` row. Match rates are exact-string on the
normalized form — deliberately, so the reported ceiling is a floor for
any fuzzy method, not an optimistic estimate.

SV incipits (§2.3) were extracted with an XML parser over the TEI
namespace, selecting `<l>` elements with `@subtype="firstline"` and
preferring `@corresp` over the rendered text. The comparison key folds
`æ/ø/å` → `ae/oe/aa`, unifies `ei`/`ej`, lowercases and strips
non-alphanumerics; matching is exact on that key, then a 24-character
prefix fallback. Register `se:` redirect entries are excluded from every
denominator, since they are aliases rather than works.

The BFN attribution in §0 was checked against the dump's own prose, the
H.C. Andersen Centre's published bibliography site, and catalogue
records — not recalled.

---

## 7b. Verification — the DDL in §4 was executed, not just written

Every statement in §4 was extracted and run against a live **PostgreSQL
16.13** server on top of `postgres-schema-design.md` §3, the parsed-works
addendum §3, and the person addendum §4–5, with `ON_ERROR_STOP=1`. The
four-document stack applies cleanly, in that order, from an empty
database. (The MySQL DDL quoted in §1 is source material, excluded from
the run.)

Behaviour-tested, not merely compiled:

| Test | Expected | Result |
|---|---|---|
| `external_title_group.is_named` on a named vs. NULL authority | computed, not asserted | `t` / `f` ✓ |
| Overlapping VID blocks (§1.2) | rejected | `EXCLUDE … int4range` ✓ |
| A second `wemi='original'` title on one work | rejected | `ewt_one_original` ✓ |
| `derived_work` + `undetermined` titles alongside the original | allowed | 3 rows ✓ |
| `'review_of'` as a derivation relation (§3.3) | **impossible** | not in `work_relation_kind` ✓ |
| Duplicate BFN number within `bfn` | rejected | `epub_bfn_number_unique` ✓ |
| Two *blank* BFN numbers (the 3 real blanks) | allowed | ✓ |
| Same-WEMI-level mapping confirmed | allowed, no reason needed | ✓ |
| **Cross-level** mapping confirmed without `verified_via` | rejected | `wem_cross_level_needs_a_reason` ✓ |
| Same, with `verified_via` | allowed, `level_agrees` computes `false` | ✓ |
| Cross-level rows left as `candidate` | unconstrained | ✓ |

Two of these carry the document's main arguments as enforcement rather
than prose: a review can never become a derivation edge (§3.3), and a
poem can never be silently confirmed as the anthology that printed it
(§2.2) — the cross-level promotion needs a human reason.

---

## 8. Phasing

| Phase | Work | Gate |
|---|---|---|
| **0** | Correct "Bjørn" → "Birger" in `hca_db_export`'s two spec documents (§0) | One-line fix; blocks nothing but misleads everyone |
| **1** | Load `external_work`, `external_work_title`, `external_publication`, `genre_vid_block` verbatim; derive genre from the VID block | No crosswalk needed; loads 100 % of both indexes |
| **2** | Load `external_publication_work` + `external_publication_about_work` — the 5,013 stated edges, `Anmeldelse af` kept separate | The BFN→Work crosswalk arrives complete (99.2 % of BFN rows) |
| **3** | ~~Parse incipits and re-match the Digte~~ — **done** (§2.3): 4.0 % → 10.8 % on the register side, 547 `hca_db`↔SV links, two candidate files emitted | Human-verify the candidates; confirm the `*` convention with the editors |
| **4** | Populate `work_external_map` at all tiers with `register_level`/`source_level` set | Zero rows promoted to `confirmed` |
| **5** | Human review, largest/most-cited works first; cross-level mappings as a separate queue | Promotion requires `verified_via`, as on the person axis |
| **6** | Resolve the 1,699 unnamed-translator rows once §9.1 is decided | Editorial decision first, code second |

---

## 9. Open questions

1. **Does the project extend the named-creator rule to unnamed
   translations?** 1,699 title rows (§3.2) are translations whose
   translator the source does not name. Under the rule as written they
   are neither new Works nor Expressions. Options: treat any translation
   as a new Work regardless of attribution; keep them as titles/aliases
   only; or research the translators. This is the largest single decision
   on this axis and it is editorial, not technical.
2. **Is `JdMs nudanske digtudgave af` (917 edges) a new Work?** Johan de
   Mylius is a named creator and modernising a text is arguably an
   adaptation — which would make it the second-largest derived-work class
   in the data. Or it is a same-creator-equivalent normalisation and
   belongs at Expression. The rule does not settle it.
3. **Should the register's WEMI level be recorded on `work` itself?**
   §2.2 shows the VÆRK-REGISTER mixes Work- and Manifestation-level
   entries with no distinguishing column. `work_external_map` records the
   level per *mapping*, which is enough for the crosswalk but leaves the
   register's own rows unclassified. A `work.wemi_level` column would fix
   that — but it is an assertion about `hca-open-repo`'s data, so it is
   the project's call, not this document's.
4. **Where does the rest of Index C enter?** (§6) `sv-datarens` gives the
   TEI side; the Laravel relational layer is still absent. Is an export
   coming, and should `sv-datarens` be registered now as a third
   `external_work_register` row with `authority_index='C'` (the column
   already allows it), keyed on `sv_xml_id`?
6. **Which SV revision is citable?** The `sv-datarens` filenames are
   working-copy names. Any confirmed incipit mapping should record the
   commit it was derived from; today that is `848532e`.
7. **What does the register's leading `*` mean?** (§2.3) 120 works carry
   it, all HCA-wing poems, undocumented. It predicts absence from SV at
   3.6 % vs 39.3 %, which is enough to route on and not enough to
   interpret. The editors will know in one sentence.
5. **`anbefalet udgave af` as editorial opinion** (§3.3) — 904 edges
   record which edition the Centre recommends. Should that surface to
   readers as a recommendation, or stay internal provenance?
