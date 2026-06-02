# Implementation plan — Bibliography export script

## 1. Goal

Provide a parameterised command-line script that selects a set of documents from
RERO ILS and exports a **structured Markdown** bibliography. The Markdown is then
converted to a **styled `.docx`** through **pandoc** (using a `reference.docx`
template), so the visual result resembles the existing *Bibliographie valaisanne*.

Two distinct deliverables are described by the client spec (`specs_bibliographies.pdf`):

- **A. Full bibliography** (pp. 1–2, 5–15): full reference template ("chablon"),
  classification-based sections ("plan de classement"), 8 indexes with active
  back-links, two-column layout, active hyperlinks to the public catalogue.
- **B. Simplified Walliser Jahrbuch bibliography** (pp. 3–4): single column,
  reduced template, **no indexes, no active links**, narrower selection. The spec
  explicitly states this variant is billed separately to the Médiathèque Valais.

The two share the same selection engine and most of the reference formatter, and
differ only in output assembly (sections/indexes/columns) and the active template.

## 2. Architecture decision

Implement a new **Flask CLI command group** registered under the existing
`reroils` entry point — *not* a standalone script — so it runs inside the app
context with access to records, the search index and config.

- Entry point: `pyproject.toml` → `[project.entry-points."flask.commands"] reroils = ...`
  (`rero_ils/modules/cli/reroils.py`). New group added via `reroils.add_command(...)`.
- New module: `rero_ils/modules/documents/bibliography/` containing:
  - `cli.py` — command + options
  - `query.py` — selection / Elasticsearch query builder
  - `reference.py` — per-document reference formatter (the "chablon")
  - `classification.py` — grouping into the per-site plan de classement
  - `indexes.py` — index builders
  - `markdown.py` — Markdown assembly (sections, two columns, anchors)
  - `config.py` — per-site plans de classement, index definitions, type
    exclusion lists, separators, label overrides.

Rationale: mirrors the project's module pattern (`api.py`/`cli.py` style), keeps
the heavy formatting logic unit-testable independently of the CLI, and lets each
library's plan de classement live in data/config rather than code.

Pandoc is invoked as an **optional post-processing step** (the command emits
`.md`; an `--docx` flag shells out to `pandoc` with `--reference-doc=` and
`--toc`). Pandoc is *not* currently a project dependency — it must be added as an
external system dependency and documented in `INSTALL.md`.

## 3. What we can reuse (verified)

Reuse the existing `_text` formatters rather than reimplementing display logic:

| Need | Reuse |
|---|---|
| title `_text` | `documents/extensions/title.py` `TitleExtension.format_text()` |
| provisionActivity `_text` | `extensions/provision_activities.py` `ProvisionActivitiesExtension.format_text()` |
| seriesStatement `_text` | `extensions/series_statement.py` `SeriesStatementExtension.format_text()` |
| editionStatement `_text` | `extensions/edition_statement.py` `EditionStatementExtension.format_text()` |
| identifier normalisation | `IdentifierFactory` / `IdentifierType` (used in `serializers/base.py`) |
| entity label per language | `authorized_access_point_fr` already present on resolved entities |
| creator-role detection | `serializers/base.py` `CREATOR_ROLES` (`aut`, `cre`, …) — spec wants `aut` only |
| permalink | `serializers/base.py::_get_permalink()` → `{get_base_url()}/global/documents/{pid}` |
| host-doc title (revues/recueils) | `dumpers/indexer.py::_process_host_document()` resolves `partOf[].document.title` |
| ref resolution | `document_replace_refs_dumper` (resolves contribution/subjects `$ref`, host docs) |

**Loading documents for export**: fetch the ES hit (already enriched by
`document_indexer_dumper`: `_text` fields, `local_fields`, resolved
`authorized_access_point_*`, `part_of[].document.title`). Where a value is missing
from the ES record, fall back to `Document.get_record_by_pid(pid).dumps(document_replace_refs_dumper)`.
Prefer the ES record to avoid one DB round-trip per document.

The `serializers/base.py::BaseDocumentFormatterMixin` is the closest existing
analogue (RIS/DC formatters) and is the right base class to subclass for the
reference formatter.

## 4. Critical open questions (must be resolved with the client before dev)

These are blockers discovered during code investigation. They drive the estimate.

1. **Local-field storage of the selection & classification.** Local fields are
   *generic* arrays `field_1..field_10` of free text
   (`local_fields/jsonschemas/.../local_field-v0.0.1.json`; ES mapping
   `documents/mappings/v7/.../document-v0.0.1.json` lines ~1536). They are **not**
   structured `$a/$b/$2` subfields. The spec's `$a valais $b 2024/09` and
   `$2 vs-bvs-sys $a xx.yy.zz` are MARC conventions — we must know **which
   `field_N` holds the selection** and **which holds the classification**, and the
   **exact string format** stored, per organisation. This determines the whole query.
2. **Date-range selection** (`valais 2024/09` → `valais 2025/08`). `local_fields`
   subfields are indexed as **`text`**, not keyword/date, so a clean range query
   is impossible at the ES level. Options: (a) match the selection token + fetch
   all hits + filter `YYYY/MM` in Python; (b) ask the client to store a normalised
   sortable token; (c) add a dedicated keyword sub-mapping (reindex required).
   Decision needed.
3. **Classification source = local field vs. document `classification` field.**
   The spec lists classification under "champs locaux", but a top-level
   `classification` field also exists (`document_classification-v0.0.1.json`, with
   `assigner` + `classificationPortion`, `assigner` indexed as `text`). Must
   confirm where each site stores it. `assigner` being `text` means exact-source
   filtering (`vs-bvs-sys`) needs a phrase/keyword strategy (possible reindex).
4. **Per-site plans de classement.** VS (`vs-bvs-sys`) is fully given in the spec.
   NE uses Dewey 23 (≤3 digits, not enumerated) and JU uses a CDU table (partial).
   Each site's complete ordered rubric tree (incl. empty rubrics to keep) must be
   delivered as config data. NE/JU trees are incomplete in the spec.
5. **Docx styling.** The target look is the 2023 Valais PDF. A `reference.docx`
   with the right styles (headings, two columns, TOC with live links, fonts,
   margins) must be built and iterated with the client. Page de couverture, page
   de titre, intro, headers/footers are explicitly **out of scope**.
6. **Index membership details.** e.g. "collectivités" excludes the
   "éditeur commercial" role; "Auteur" index lists **all** contributors/roles and
   replaces the title by the internal reference number. These per-index rules need
   confirmation against the field-source table (spec p. 2).

## 5. Detailed work breakdown

### 5.1 CLI + configuration infrastructure
- New command group `reroils bibliography export` with options:
  `--organisation/--library`, `--selection` (token, e.g. `valais`),
  `--from`/`--to` (e.g. `2024/09`), `--classification-scheme` (`vs-bvs-sys` …),
  `--type`/`--exclude-type`, `--language`, `--profile` (`full` | `walliser`),
  `--output`, `--docx`, `--reference-doc`, `-v/--verbose`.
- `config.py` holding per-site plans de classement (ordered rubric list with
  code + FR/DE labels), index definitions, the type/subtype **exclusion list**
  (spec p. 4), "classement au titre" type list (spec p. 6), and label overrides
  (e.g. `docsubtype_documentary` → "film documentaire").

### 5.2 Selection / query engine (`query.py`)
- Build a `DocumentsSearch` filter from options: selection-token match on the
  configured local field, type `main_type`/`subtype` include/exclude
  (`type.main_type`/`type.subtype` are `keyword` — exact filters), language
  (`language.value` is `keyword`).
- Implement the date-range selection per the decision in §4.2.
- `scan()` iteration with `click.progressbar`.

### 5.3 Reference formatter (`reference.py`) — the "chablon"
Subclass `BaseDocumentFormatterMixin`. Implement the ordered template:
`type` line → `authorized_access_point_fr. – title._text / responsibility. –
edition. – sequence_numbering. - scale. - provision, copyright. – extent
(duration): illustrativeContent ; dimensions. – (series)` → `ID:` line →
`note. – dissertation. – credits. – tableOfContents. – frequency` → electronicLocator (full profile only).

Rules to implement (spec pp. 5–6):
- **Conditional punctuation**: when a field is null, suppress its associated
  punctuation and the separator to the previous/next value. This is the core
  fiddly logic — build a small "segment join" helper that drops empty segments
  and their delimiters.
- **author**: first contributor whose `contribution.entity.role` contains `aut`,
  shown via `authorized_access_point_fr`.
- **Per-field multi-value separators**: `responsibility`/`edition`/`sequence_numbering`/`scale` → ` ; `;
  `copyrightDate`/`duration`/`illustrativeContent`/`dimensions` → `, `;
  `seriesStatement` → ` ` with each series in its own parentheses;
  identifiers/notes/dissertation/credits/tableOfContents → `. - `;
  `frequency` → ` ; `.
- **Identifier cascade**: pick the first existing type in order
  Ean → Isbn → Issn → Ismn → Doi → Upc → PublisherNumber; print all of that type.
- **Type display**: subtype if present else main_type; apply label overrides.
- **notes**: exclude `cited_by` note type.
- **electronicLocator**: keep only `resource` URLs (full profile).
- **"classement au titre"** types (spec p. 6) and anonymous docs: omit the
  leading author and start at the title.
- **RERO+ pid line** with permalink + numerus currens number (1..X).

### 5.4 Classification grouping (`classification.py`)
- Read each document's classification value (per §4.3 decision), map to a rubric
  in the site's ordered plan de classement.
- Emit rubrics in numeric order; keep empty rubrics; documents under their rubric.
- Handle the two-level FR/DE rubric headings.

### 5.5 Index generation (`indexes.py`) — full profile only
Eight indexes (spec pp. 1–2, 7): systématique, géographique, biographique,
sujets, collectivités, auteur, anonymes, revues. For each:
- Source field per the spec table (`subjects.entity` filtered by `bf:Place` /
  `bf:Person` / `bf:Topic` / `bf:Organisation` and `authorized_access_point_fr`;
  `contribution.entity` for auteur; `title._text` for anonymes;
  `partOf.document` for revues).
- Each entry links (Markdown anchor) back to its reference number.
- Apply per-index role rules (e.g. collectivités excludes "éditeur commercial").

### 5.6 Markdown assembly (`markdown.py`)
- Emit a TOC (active links to sections), classification sections, references with
  stable anchors (`{#ref-N}`), and indexes (entries link to `#ref-N`).
- Two-column layout is a **pandoc/docx concern**, expressed via the
  `reference.docx` template + a pandoc div/column attribute, not in raw Markdown.

### 5.7 Pandoc / docx
- Build `reference.docx` matching the 2023 Valais styling (iterative with client).
- `--docx` flag shells out: `pandoc out.md --reference-doc=reference.docx --toc -o out.docx`.
- Document the pandoc dependency in `INSTALL.md`.

### 5.8 Simplified Walliser Jahrbuch profile (B — billed separately)
- Reduced chablon (no electronicLocator, single column, no indexes, no links).
- Selection: monographs + audio only, German language (audio any language),
  treatment-date window. Reuses the same engine with a `walliser` profile.

## 6. Tests (TDD, function-based, `tests/`)
- `tests/unit/documents/bibliography/`:
  - reference formatter: one test per chablon rule incl. **null-punctuation
    suppression**, identifier cascade, author selection, type label override,
    classement-au-titre, multi-value separators.
  - query builder: selection token, date window, type include/exclude, language.
  - classification grouping: ordering, empty rubrics kept, FR/DE headings.
  - index builders: membership rules, anchors/back-links, role exclusions.
- `tests/api/` or `tests/ui/`: end-to-end CLI run on fixtures producing Markdown;
  assert structure (sections, ref numbers, index links). Add fixtures with local
  fields, classification, partOf host docs, linked + local entities.
- Use existing fixtures (`tests/data/local_fields.json`, document fixtures);
  extend minimally. Do **not** test pandoc itself — only that we invoke it / emit
  valid Markdown.

## 7. Success criteria
1. `reroils bibliography export --profile full …` produces Markdown that pandoc
   renders to a two-column docx matching the agreed styling, with working TOC and
   index back-links.
2. Selection, classification grouping, and every chablon punctuation rule are
   covered by passing unit tests.
3. `--profile walliser` produces the simplified single-column document.
4. `uv run poe lint` and `uv run poe format` clean.

## 8. Risks
- **Local-field convention** (§4.1/4.2): if storage is unstructured/inconsistent
  across records, selection accuracy suffers and may need a reindex or data
  cleanup (out of this scope — flag to client).
- **`assigner`/local-field `text` mappings** make exact filtering imprecise; may
  require a mapping change + full document reindex.
- **Per-site plans de classement** (NE Dewey, JU CDU) are incomplete in the spec;
  blocked on client data.
- **Docx fidelity** to the 2023 PDF is inherently iterative.
