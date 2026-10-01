# Build Brief: `application_intelligence` Module

**Audience:** Implementing AI assistant (Astra). **Status:** Design approved with revisions. This brief is the authoritative instruction set — where it differs from earlier exploration notes, this brief wins. **Parent product:** NIC AI PREMIA Support Intelligence Platform. This module is the first implementation of its "Application Intelligence" capability. It must honor the platform's trust model:

```
Deterministic analysis = SOURCE OF TRUTH
LLM / AI              = REASONING + HYPOTHESIS (never structural authority)
```

---

## 1. What you are building

A self-contained Python module, `application_intelligence/`, that ingests Oracle Forms artifacts (FMB), converts them to XML, extracts a deterministic, evidence-backed structural knowledge graph, adds a strictly separated AI semantic layer, and answers natural-language questions with citations that trace to exact evidence.

Scope for this phase: a working pipeline proven on two forms (`FM2055_OICL`, `FT2059_OICL`). Architecture must not preclude scaling to the full corpus (\~1,000 forms, \~138K items, \~58K triggers).

## 2. Hard constraints (non-negotiable)

1. **Isolation.** Zero files outside `application_intelligence/` are created or modified. No writes to `nic_ai.db` or any existing store. The only external import is `services/llm_provider.py`, read-only, imported inside exactly one lazy adapter module.
2. **No fabricated structure.** The LLM may never create facts or edges. It produces claims only, each citing existing evidence IDs; claims citing unknown IDs are rejected at validation.
3. **Determinism degrades gracefully.** If the LLM/embeddings are unconfigured or unavailable, the deterministic pipeline must still complete end to end; the run is marked `llm_not_configured`.
4. **Provenance everywhere.** Every fact, edge, and AI claim foreign-keys to an evidence row (artifact sha256 + XML xpath + line range + exact quote).
5. **Environment:** Python 3.11.9, Windows, corporate TLS proxy (`pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org`). pytest 9.1.1, numpy/scipy available. Prefer stdlib; every third-party dependency must be pinned in `application_intelligence/requirements.txt` and justified.
6. **Tests never touch Oracle or the network.** Fixtures and fakes only; the real conversion test auto-skips when `ORACLE_HOME` is absent.
7. New env vars use the `AI_*` prefix; the LLM adapter reuses `RCA_LLM_*` when set.

## 3. Module layout

```
application_intelligence/
  __init__.py  config.py  pipeline.py  README.md  requirements.txt
  ingestion/     artifact_store.py   fmb_converter.py   xml_parser.py
  understanding/ models.py  evidence.py  extractor.py  relationships.py
                 ai_analysis.py  prompts.py
  knowledge/     knowledge_base.py      # facade over the stores
  visual/        visual_store.py        # linkage only; no renderer
  retrieval/     chunking.py  embedder.py  vector_index.py
  query/         retrieval.py  reasoning.py  cli.py
  storage/       store.py  paths.py     # single SQLite DB + raw file dir
  tests/         fixtures + fakes
```

## 4. Storage: ONE SQLite database (decided — do not split)

Single DB file so that facts, edges, claims, and chunks can enforce real foreign keys into `evidence`, and so an ingestion run is atomic. Raw artifact bytes are copied to `application_intelligence/storage/artifacts/<form>/` (NOT an `input/` directory). That directory and the DB file must be gitignored.

Seven table families:

1. **artifacts** — sha256, size, mtime, origin path, kind (FMB/FMX/XML), conversion status, conversion stderr/warnings, `dataset_id`, `pipeline_version`. **Immutability rule:** a re-conversion produces a NEW artifact row (new sha256); never mutate an existing row, or evidence line ranges silently break.
2. **facts\_*** — Form, Window, Canvas, TabPage, Block, Item, Trigger, ProgramUnit, LOV, Relation. Deterministic only. Every row: `evidence_id`, `classification`, `pipeline_version`, `dataset_id`.
3. **ai_claims** — subject ref, predicate, text, model, prompt_version, scope, timestamp, accepted/rejected, cited `evidence_id[]`. Never mixed into facts tables.
4. **edges** — typed, directed, `resolved` flag, `evidence_id[]`, `source` (`deterministic` | `ai`), `classification`.
5. **vec_chunks** (sqlite-vec vec0 virtual table) + **chunks** metadata — chunk kinds: item summary, trigger code, program unit, form overview, accepted semantic claim. Each chunk records `embedding_model`.
6. **visual_assets** — linked to entity ids; ingestion path only for now.
7. **evidence** — artifact sha256 + xpath + line range + exact quote.

**Classification enum (new, required):** every fact, edge, and claim carries one of `OBSERVED | DERIVED | HYPOTHESIS | RUNTIME_CONFIRMED | UNRESOLVED`. Parsed-from-XML facts are OBSERVED; computed edges (e.g., resolved calls) are DERIVED; all AI claims are HYPOTHESIS. RUNTIME_CONFIRMED is reserved, unused in this phase.

**Scoping (new, required):** every artifact and derived row carries `dataset_id`. One hardcoded value is fine for this phase; the column exists from day one so multi-customer isolation is not a retrofit.

**Versioning (new, required):** `pipeline_version` (extractor version) on deterministic rows, mirroring `prompt_version`/`model` on claims.

## 5. Relationship model (typed edges)

- Form→CONTAINS→Window; Window→CONTAINS→Canvas; Block→CONTAINS→Item
- **Block→DISPLAYED_ON→Canvas (many-to-many, derived per-item from CanvasName).** Do NOT model Canvas→CONTAINS→Block — a block's items span multiple canvases; containment implies a single parent and is wrong here. Item→belongs-to→Block and Item→displayed-on→Canvas are the ground truths; the block↔canvas edge is an aggregation.
- Item/Block/Form→HAS_TRIGGER
- Trigger→CALLS→ProgramUnit (PL/SQL call parsing); ProgramUnit→CALLS→Procedure/Function (`resolved=false` with external name when unresolved)
- ProgramUnit/Block→READS|WRITES→DatabaseObject (SQL DML/SELECT + QueryDataSourceName)
- Form→NAVIGATES_TO→Form|Block (OPEN_FORM/CALL_FORM/GO_BLOCK); Item→NAVIGATES_TO→Item (Next/PreviousNavigationItemName)
- Block→MASTER_DETAIL→Block (Relation)

Boundary note: this module produces structural facts and dependency edges. Control-flow/CFG analysis remains the existing RCA engine's domain — do not reimplement it; leave a named integration point.

## 6. Ingestion flow and frmf2xml handling

input files → copy + sha256 → frmf2xml per form (failure-isolated) → XML parse → facts + evidence + edges → stage markers keyed by sha256 (idempotent re-runs).

`fmb_converter.py` requirements:

- Clear `JAVA_HOME` in the subprocess environment (JDK 24 crashes frmf2xml; Oracle's bundled JDK 8 must be used).
- Exit code -1 with valid XML output = `success_warnings`, not failure.
- **Capture and store stderr** on the artifact row.
- **Sanity-check before marking success:** XML is well-formed AND counts of windows, blocks, and items are each > 0. Exit -1 with subtly truncated XML must not pass silently.

## 7. AI analysis flow (revised: chunked, not one call)

One call per form does not survive real forms (FM2055 has 555 triggers, 32 program units). Instead:

1. Build a compact deterministic structural summary (JSON) per form — shared context.
2. Run **per-scope analysis calls** (per block, or per trigger-cluster) with that shared context plus the scope's actual code.
3. Optional roll-up call for the form overview.
4. Every response: schema-validated JSON; every claim must cite existing evidence IDs; unknown IDs → claim rejected and logged.
5. Cache key: `(artifact sha256, scope, prompt_version, model)`.
6. Store as `ai_claims` with model + prompt_version + scope. Accepted/rejected is a real workflow column — citation validity ≠ truth; a claim can cite real evidence and still misread it, so the two-form experiment must include a manual spot-check of a sample of claims before the semantic layer is trusted.

## 8. Retrieval/query flow

question → embed (OpenRouter `/embeddings` via the adapter, cached) → hybrid search (sqlite-vec kNN + SQLite FTS5) → 1–2 hop graph expansion over edges → evidence-grounded context pack within token budget → LLM reasoning → answer with citations resolving to evidence rows.

`vector_index.py` requirements:

- sqlite-vec pinned to an exact version (it is pre-1.0; expect API churn).
- **Pure-numpy brute-force fallback behind the same interface** — `enable_load_extension` may be compiled out on some Windows Python builds, and on-prem customer environments may forbid the extension. At two-form scale the fallback is fully sufficient.
- Each chunk stores its `embedding_model`; a model change marks the index stale.

## 9. Testing

`python -m pytest -q application_intelligence/tests`

- Synthetic Forms-XML fixture: parser, relationship (including Block→DISPLAYED_ON→Canvas many-to-many), and evidence tests.
- Fake LLM client (recorded JSON) + deterministic fake embedder for AI/index tests, including the unknown-evidence-ID rejection path and the `llm_not_configured` graceful-degradation path.
- Converter tests: JAVA_HOME clearing, exit -1 + valid XML → success_warnings, sanity-check rejection of truncated XML (use a fixture, not Oracle).
- Real conversion test auto-skips without `ORACLE_HOME`.

## 10. Acceptance: two-form experiment

```
pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org -r application_intelligence/requirements.txt
set ORACLE_HOME=D:\Oracle\Middleware\Oracle_Home
<run set_llm_env.ps1>
python -m application_intelligence.pipeline run --input <path> --forms FM2055_OICL FT2059_OICL
python -m application_intelligence.query.cli "How does this form navigate between screens?"
python -m pytest -q application_intelligence/tests
```

Pass criteria: all tests green; both forms ingested with facts/edges/evidence populated and FM2055 roughly matching the proven conversion (≈21 blocks, 702 items, 555 triggers, 32 program units, 19 canvases, 3 windows); the CLI answer cites evidence rows that resolve to real xpath/line ranges; re-running the pipeline is a no-op (idempotency by sha256); pipeline completes deterministically with the LLM disabled.

## 11. Process

Follow the platform's implementation discipline: produce a written specification from this brief for human review **before writing code**, then an implementation plan broken into tasks, then TDD per task. Do not expand scope (no CFG, no renderer, no multi-customer UI, no runtime database access). When this brief is ambiguous, ask; do not invent.