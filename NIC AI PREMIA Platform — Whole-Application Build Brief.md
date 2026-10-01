# Build Brief: NIC AI PREMIA Support Intelligence Platform (Whole Application)

**Audience:** Implementing AI assistant (Astra). **Scope:** The complete platform — not only the Application Intelligence module. This brief turns the master product document into a buildable, phased engineering plan. **Authority:** The master product document ("NIC AI PREMIA Support Intelligence Master Product") defines WHAT; this brief defines HOW and IN WHAT ORDER. Where the two conflict, raise it — do not silently choose. **Process rule (from the master doc, binding):** This brief does not authorize skipping the approved flow. Each phase below must produce: Specification → Implementation Plan → Tasks → TDD → Targeted tests → Full regression → Security/boundary verification → Milestone review → Acceptance. Produce the Phase 1 specification for human review before writing any code.

---

# 1. What the product is

An evidence-driven support investigation platform for PREMIA. A support engineer goes from a Jira incident to an evidence-backed RCA by connecting: ticket intelligence → PREMIA area identification → Application Intelligence (Forms/XML, blocks, items, triggers, program units, dependencies, CFG, errors) → Database Intelligence → deterministic evidence → bounded AI reasoning → deterministic validation → optional read-only runtime evidence → SME review → support decision.

It is NOT a generic chatbot, not an autonomous agent, not a system where AI output is treated as fact.

## 1.1 Trust model (governs every design decision)

```
Deterministic analysis = SOURCE OF TRUTH
LLM / AI              = REASONING + HYPOTHESIS
Runtime evidence      = CONFIRMATION
SME                   = FINAL VALIDATION
```

## 1.2 Non-negotiable principles (enforced in code and review)

1. Deterministic analysis is authoritative; LLM reasoning is bounded by supplied evidence.
2. No fabricated source, dependency, or runtime facts. AI claims must cite evidence IDs; claims citing unknown IDs are rejected at validation.
3. Every important finding carries provenance (artifact + version + location + quote).
4. Every finding carries a classification: `OBSERVED | DERIVED | HYPOTHESIS | RUNTIME_CONFIRMED | UNRESOLVED`. A hypothesis is never presented as fact.
5. Runtime database access is read-only, explicit, auditable. No production mutation, ever.
6. Customer/environment isolation is mandatory; every query path is tenant-scoped.
7. Ticket text, comments, attachments, and screenshots are UNTRUSTED input — evidence, never instructions. They are quarantined from prompts as directives and sanitized in UI.
8. No credentials in prompts, logs, evidence, screenshots, or audit output.
9. Indexes, analysis, prompts, models, and knowledge are versioned; an RCA is reproducible against the versions that produced it.
10. Modular monolith first; extract services only when operations demand it. Do not over-engineer.
11. The existing deterministic RCA engine is the foundation — build around it, never replace it with an LLM, never break it.

---

# 2. Target architecture

## 2.1 Shape

**One modular-monolith backend + one web frontend + background job workers + per-tenant data stores.** No microservices, no message brokers, no Kubernetes in this phase. Async jobs run via a simple DB-backed job queue (table + worker process) — observable states: `RUNNING | WAITING | COMPLETED | FAILED | PARTIAL | BLOCKED`, with progress percentages for long operations.

## 2.2 Backend layering

```
/interfaces       HTTP API (REST, JSON), CLI, job worker entrypoints
/application      use cases / orchestration (one class per user-facing operation)
/domain           entities, evidence model, classification rules, invariants
/infrastructure   SQLite stores, Jira client, Oracle client, frmf2xml wrapper,
                  LLM adapter, embedder, file storage, secrets
```

Dependency direction: interfaces → application → domain; infrastructure implements domain-defined ports (Protocol classes). Domain imports nothing from infrastructure.

## 2.3 Backend modules (bounded, each with public interface + test boundary)

```
tenancy          customers, environments, configuration, dataset versions
security         authN (session), RBAC, secrets, audit log
jira             Jira client, ticket sync, comments/history/attachments
ticket_intel     signal extraction (error codes, policy/claim numbers, form/
                 block/item/trigger names, tables, packages, PREMIA concepts)
attachments      storage, PDF/doc/image handling, screenshot evidence linkage
artifacts        artifact registry, FMB→XML conversion, checksums, versions
forms_intel      Forms XML parsing → facts (form/window/canvas/block/item/
                 trigger/program unit/LOV/relation) + evidence
app_intel        typed dependency edges, call resolution, DB references,
                 navigation edges  (the module briefed previously — reuse it)
cfg              control-flow analysis (wrap/extend EXISTING engine; do not rewrite)
db_intel         Oracle metadata sync (tables/columns/packages/procedures),
                 app→DB reference joins, read-only runtime queries
sql_chatbot      NL→SQL proposal → safety validation → read-only execution → explanation
knowledge        PREMIA knowledge: common / customer / environment, with origin tags
retrieval        FTS5 + vector hybrid search, graph expansion, context packing
rca              sufficiency gate, bounded context, LLM reasoning, deterministic
                 validation of claims, RCA record assembly
runtime_evidence read-only diagnostics, execution audit, static/runtime separation
sme_review       review queue, accept/reject/needs-investigation, promotion of
                 validated outcomes to trusted knowledge/eval data
evaluation       golden dataset, retrieval/RCA accuracy metrics, unsupported-claim rate
observability    request IDs, latency metrics, job metrics, failure rates
```

Avoid god modules. `app_intel` is the module from the earlier brief — it slots in under this architecture with its storage conventions intact (single SQLite evidence-centric store, classification enum, dataset_id, pipeline_version).

## 2.4 Frontend

React + TypeScript single-page app (confirm stack with owner; this is the working assumption implied by the master doc's /components /hooks /services layout).

```
/tickets        ticket list, ticket workspace (summary, comments, history,
                attachments, extracted signals, RCA status, SME status)
/forms          Form Explorer (tree: blocks→items, triggers, program units,
                dependencies, DB refs, errors, CFG) + XML/source explorer
                with citations
/dependencies   graph navigation over typed edges
/database       table/column/package exploration, app→DB references
/rca            RCA workspace: evidence panels, AI hypothesis (clearly labeled
                HYPOTHESIS), missing evidence, recommended validation
/runtime        runtime evidence requests + results (read-only), audit trail
/knowledge      PREMIA knowledge browser with origin (common/customer/env)
/evaluation     golden dataset, metric dashboards
/admin          customers, environments, onboarding wizard, health, roles
shared: /components /hooks /services /types /utils
```

Every UI surface that shows AI output must visually distinguish classification (OBSERVED / DERIVED / HYPOTHESIS / RUNTIME_CONFIRMED / UNRESOLVED) and link every important element to its evidence/source citation.

## 2.5 Data stores

- **Platform DB (SQLite per deployment):** tenancy, users/roles, audit, jobs, Jira sync state, SME reviews, evaluation.
- **Per-environment intelligence DB (SQLite):** artifacts, facts, edges, evidence, ai_claims, chunks + vectors (sqlite-vec with numpy fallback), knowledge, db metadata, runtime evidence records. One file per customer-environment-dataset ⇒ isolation is physical, not just a WHERE clause.
- **File store:** `storage/<customer>/<environment>/<dataset>/artifacts|attachments|screenshots/…`, originals immutable, addressed by sha256.
- SQLite is the committed store for this phase (single-writer job model makes it safe). Keep store access behind repository interfaces so Postgres/Oracle can replace it later without touching domain code.

## 2.6 Environment constraints

Python 3.11.9 on Windows; stdlib-first culture; every dependency pinned and justified; corporate TLS proxy (`pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org`); pytest 9.1.1; numpy/scipy available. frmf2xml requires `JAVA_HOME` cleared (uses Oracle's bundled JDK 8) and exits -1 while producing valid XML — treat exit -1 + well-formed XML + nonzero window/block/item counts as `success_warnings`, record stderr. LLM/embeddings via the existing `services/llm_provider.py` abstraction (OpenRouter today), env `RCA_LLM_*` / new `AI_*`. On-prem/local LLM must remain possible — never hardcode a provider.

---

# 3. Core cross-cutting designs

## 3.1 Evidence & provenance model (shared by all modules)

`evidence` rows: source kind (XML xpath+lines, ticket field, attachment+page/region, DB metadata query, runtime query), artifact sha256 / record ref, exact quote, captured-at. Facts, edges, ai_claims, RCA findings, and runtime results all FK to evidence. An RCA record stores: who, when, customer, environment, ticket, dataset version, artifact versions, RCA engine version, CFG version, prompt version, model, supplied evidence set, validation accept/rejects, SME decision — the full section-32 audit answer set.

## 3.2 AI boundary (shared)

LLM calls receive only: ticket intelligence + deterministic evidence + scoped PREMIA knowledge + optional runtime evidence, inside a token-budgeted context pack built by `retrieval`. Output is schema-validated JSON; each claim cites evidence IDs; the deterministic validator rejects unknown citations and flags contradictions; everything stored as HYPOTHESIS until SME action. A sufficiency gate runs before reasoning: if evidence is insufficient/ambiguous, the system says so (UNRESOLVED path) instead of calling the LLM. LLM unavailable ⇒ all deterministic surfaces still work.

## 3.3 Untrusted-input boundary

Ticket/attachment/screenshot text is tagged untrusted at ingestion. It may be displayed, indexed, and quoted as evidence; it is never interpolated into prompts as instructions (structured fields only, with explicit "untrusted content" framing), never used to choose tools/queries without deterministic validation, and never allowed to inject SQL (sql_chatbot validates against metadata + read-only allowlist regardless of question text).

## 3.4 SQL Chatbot safety

NL question → SQL proposal (LLM, schema-grounded) → deterministic validation: parse, single-statement, SELECT-only allowlist, object names must exist in synced metadata, row/time limits, bind variables for user values → execute via dedicated read-only Oracle account → result + explanation; proposal, validation verdict, and execution all audited. Generated SQL is shown to the user before/with execution; it is never trusted implicitly.

## 3.5 Failure states

Every flow surfaces explicit reasons (master doc §33): Jira down/auth failure, attachment/image failure, no/multiple candidate forms, insufficient or ambiguous evidence, CFG unavailable, DB unavailable, LLM unavailable/validation failure, runtime not executed/error, stale index, failed conversion, missing source/dependency, SME pending. "Why can't the system proceed?" must always be answerable from the UI.

## 3.6 Incremental processing & versioning

Change detection by sha256; reprocess only changed artifacts; update affected dependencies, CFG, and index entries. Dataset version increments on ingestion; analyses record the versions they ran against. No full-corpus reprocessing for a one-form change.

---

# 4. Phased delivery plan

Each phase ends usable and demoable; later phases never break earlier acceptance tests.

## Phase 0 — Platform skeleton (foundation)

Backend layering, tenancy (customer/environment/dataset, single seeded tenant "NIC"), config, secrets handling, session auth + RBAC roles (Platform Admin, Customer Admin, Support Engineer, Developer, DBA, SME, Viewer, QA), audit log, job queue + worker, observability (request IDs, timings), frontend shell with auth + /admin. **Accept:** login, roles enforced on a dummy resource, a job runs with visible RUNNING→COMPLETED states, audit rows written.

## Phase 1 — Artifact & Application Intelligence core

Artifact registry + FMB→XML conversion (converter rules above), forms_intel extraction, app_intel typed edges, evidence store, FTS5 + vector index, query CLI/API. This is the previously-briefed `application_intelligence` module integrated under /infrastructure+/domain, proven on FM2055_OICL + FT2059_OICL (expected shape ≈ 21 blocks, 702 items, 555 triggers, 32 program units, 19 canvases, 3 windows for FM2055). Relationship model per earlier brief, including Block→DISPLAYED_ON→Canvas (many-to-many — NOT canvas-contains-block). **Accept:** earlier module acceptance + Form Explorer renders both forms with citations; idempotent re-ingestion; pipeline completes with LLM disabled.

## Phase 2 — Jira & Ticket Intelligence

Jira connector (configurable per environment), ticket sync (fields, comments, history, attachments), ticket workspace UI, ticket_intel extraction of signals (error codes, policy/claim numbers, form/block/item/trigger/table/package/procedure names, PREMIA concepts) — signals, not facts. Attachment store; image/PDF text where feasible; screenshots linked as visual evidence. **Accept:** open a real ticket, see extracted signals with sources, jump from a form-name signal into Form Explorer; Jira-down and auth-failure states render correctly.

## Phase 3 — RCA v1 (static)

Sufficiency gate, bounded RCA context (ticket intel + retrieval over Phase 1 evidence + knowledge), LLM reasoning per §3.2 with chunked per-scope analysis where needed, deterministic claim validation, RCA workspace UI producing the master-doc §23 output shape (relevant form/item/trigger, dependency evidence, DB references, errors, static citations, clearly-labeled AI hypothesis, missing evidence, recommended validation, Runtime: NOT_PERFORMED, SME: REQUIRED). SME review queue with Accepted/Rejected/Needs-Investigation. **Accept:** end-to-end ticket→RCA on a seeded ticket; every hypothesis cites resolvable evidence; a claim citing a fake ID is rejected and visible as rejected; insufficient-evidence ticket yields UNRESOLVED, not a confabulated RCA.

## Phase 4 — Database Intelligence & SQL Chatbot

Read-only Oracle metadata sync (tables, columns, packages, procedures/functions), app→DB reference joins (edges from Phase 1 resolved against real metadata), /database explorer, sql_chatbot per §3.4. **Accept:** "Does this claim exist?" style questions execute safely with visible SQL + audit; DDL/DML attempts are blocked at validation; metadata is tenant-scoped.

## Phase 5 — Runtime evidence & RCA v2

Explicit, auditable read-only runtime diagnostics attached to an RCA; Runtime status CONFIRMED/CONTRADICTED/NOT_PERFORMED; static vs runtime evidence visually and structurally separate; CFG integration surfaced in Form Explorer and RCA context (wrapping the existing engine). **Accept:** an RCA upgraded from HYPOTHESIS toward RUNTIME_CONFIRMED by an executed, audited read-only check; a contradicting runtime result flags the hypothesis.

## Phase 6 — Knowledge, onboarding, evaluation, scale-out

PREMIA knowledge store (common/customer/environment with origin tags; SME-validated outcomes promoted per master doc §29), 13-step onboarding wizard with per-step status/validation (master doc §8), environment health view (§9), golden dataset + evaluation metrics (form/item/trigger/DB-object retrieval accuracy, RCA accuracy, unsupported-claim rate, latency), incremental corpus ingestion toward the full \~1,091 forms, second-customer dry run proving isolation. **Accept:** onboarding a fresh environment end-to-end via wizard; evaluation run produces the metric report; cross-tenant access attempts fail at every API.

---

# 5. Testing & quality bar

- TDD per task; unit tests at domain level with fakes (fake Jira, fake Oracle, fake LLM with recorded JSON, deterministic fake embedder); no network/Oracle in the default test run; conversion tests auto-skip without `ORACLE_HOME`.
- Per-phase integration tests + a growing end-to-end regression suite (`python -m pytest -q`).
- Security tests as first-class: tenant-isolation tests on every API, prompt-injection tests (hostile ticket text attempting instruction injection and SQL injection), secrets-never-logged assertions, read-only enforcement tests for the Oracle account path.
- Evaluation harness (Phase 6) runs against the golden dataset and reports unsupported-claim rate — the product's core quality metric.

# 6. Explicit non-goals (do not build)

Autonomous production changes; write access to any customer database; an unrestricted agent loop; LLM-chosen retrieval over raw source; replacing the existing RCA engine or CFG analyzer; microservices/queues/k8s; mixing AI claims into fact tables; cross-customer anything.

# 7. Open decisions to confirm with the owner before Phase 0 coding

1. Frontend stack confirmation (React+TS assumed) and packaging (served by the monolith vs separate dev server).
2. Backend HTTP framework (suggest FastAPI; stdlib-only is impractical for a full web app — needs explicit approval as a dependency, pinned).
3. AuthN source (local accounts vs corporate SSO/LDAP).
4. Where the existing RCA engine's code/DB lives relative to the new monolith, and the integration contract for wrapping (not rewriting) it and the CFG analyzer.
5. Jira instance type (Cloud vs Server/DC) and credential model per environment.

# 8. First deliverable

Produce the **Phase 0 + Phase 1 specification** (not code): module interfaces, DB schemas, API endpoints, UI screens, test plan, and the §7 decisions as explicit questions — for human review and approval. Then plan → tasks → TDD.