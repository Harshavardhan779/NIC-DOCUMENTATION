# PREMIA Support Intelligence Platform — Independent Product & Architecture Proposal

*An AI-assisted, evidence-governed root-cause-analysis product for PREMIA / Oracle-based insurance estates.*

**Document status:** Proposal for comparison against an alternative architecture **Author role:** Senior enterprise product / UX / systems architect **Convention used throughout:** claims are tagged as **REQUIREMENT**, **DESIGN DECISION**, **ASSUMPTION**, **TRADE-OFF**, or **FUTURE OPTION**.

---

## 1. Executive Summary

The product proposed here — working name **Tracer** — is an on-premise-first investigation platform that turns a Jira support ticket into an evidence-backed root-cause analysis of a PREMIA (Oracle Forms + PL/SQL + Oracle DB) estate, with a human expert in the loop at every consequential step.

Five architectural pillars define the design, and everything else follows from them:

1. **Isolation by deployment, not by policy.** Each customer runs their own self-contained deployment ("cell") inside their own network. Customer source code, database metadata, and Jira content never leave the customer boundary. Multi-tenancy exists only in a thin, optional vendor control plane that carries licensing, product telemetry (opt-in), and software updates — never customer artifacts. This converts the hardest requirement (strict isolation of insurance source code and data) from an ongoing policy-enforcement problem into a structural property.
2. **A deterministic Code Knowledge Graph (CKG) is the single source of truth.** Forms, FMBs, PLLs, reports, PL/SQL, and database catalog metadata are parsed by deterministic pipelines into a typed graph of artifacts, symbols, and relationships stored in PostgreSQL. The LLM never reads the filesystem or the database directly; it can only call tools that query the CKG and return real, versioned, addressable evidence.
3. **Evidence is a first-class data model, not a prose convention.** Every finding in an RCA is a typed claim linked to evidence records (Observed / Derived / Hypothesis / Unresolved / Runtime-confirmed). A deterministic **claim verifier** checks every citation before anything is shown to a user; uncited claims are automatically downgraded to hypotheses and visually flagged. This is how "the AI must not invent" is enforced mechanically rather than by prompt-wishing.
4. **A modular monolith with an async job plane.** One deployable application (API + UI) plus a pool of workers sharing a job queue built on PostgreSQL. No Kubernetes requirement, no Kafka, no Neo4j. The scale envelope (tens of thousands of artifacts, thousands of tickets, tens of concurrent users, per customer) fits comfortably in this shape, and it is the only shape that support-team-sized customers can realistically operate on-premise.
5. **Evaluation is built into the core schema from day one.** Completed, SME-validated investigations *are* golden records. The pipeline is replayable stage-by-stage against them, so retrieval accuracy, RCA accuracy, and unsupported-claim rate become regression metrics rather than a bolted-on QA activity.

The MVP is deliberately narrow: one Jira connector, Forms-XML + PL/SQL + DB-catalog ingestion, hybrid retrieval, an agentic investigation with a bounded tool budget, the RCA workspace with the evidence ledger, and SME validation. Runtime verification, vector search, cross-ticket learning, and additional connectors are staged afterwards.

---

## 2. Problem Analysis

### 2.1 What the problem actually is

Restated from first principles, the job is: *given an under-specified, noisy description of a failure (a ticket), locate the small set of code and data objects that explain the failure, and produce a defensible causal narrative connecting symptom to cause.*

Three properties of this domain shape everything:

**The investigation is a graph traversal, not a text search.** The path from "user gets ORA-20101 on claim settlement screen" to "package `PKG_CLM_SETTLE.VALIDATE_LIMITS` reads a stale row in `CLM_COVER_LIMITS` because trigger `WHEN-VALIDATE-ITEM` on block `SETTLE_DTL` bypasses the refresh" runs through explicit, machine-recoverable relationships: form → block → item → trigger → program unit → package → table. Humans traverse this graph slowly and from memory. A machine can traverse it precisely — **if the graph exists**. Therefore the core asset of the product is not the LLM; it is a correct, current, queryable dependency graph of the customer's estate. The LLM's job is to decide *which* traversals to run and to synthesize the result into a narrative.

**The cost of a wrong answer is asymmetric.** A support engineer who receives a confidently wrong RCA will spend hours disproving it and will stop trusting the tool after two or three such events. A tool that says "here are the 4 relevant artifacts, 2 confirmed facts, and 1 hypothesis I could not confirm" is more valuable than one that always produces a full narrative. This drives the evidence-ledger design and the decision to make "Unresolved" a first-class, prominently displayed outcome rather than a failure state.

**PREMIA estates are heavily customized and version-fragmented.** No two customers share a schema, form set, or vocabulary. So the product cannot ship with a hardcoded PREMIA model. It ships with (a) parsers for the *artifact formats* (which are stable Oracle formats: XML from frmf2xml, PL/SQL grammar, data dictionary views), and (b) a curated but *versioned and per-customer-overridable* domain knowledge base.

### 2.2 Why current practice fails

Investigation today is bottlenecked on a few SMEs whose value is precisely the mental dependency graph and error-code vocabulary they carry. Their knowledge is unindexed, unversioned, and leaves when they do. Junior engineers cannot even formulate the right `grep`. Screenshots — often the richest evidence in a ticket — are unusable to any text tooling.

### 2.3 Hidden requirements the statement implies but does not say

- **Index freshness.** Customers patch PREMIA and deploy customizations continuously. An RCA computed against last month's index is dangerous. → incremental re-indexing and *snapshot pinning* (every investigation records which index snapshot it ran against) are core requirements, not optimizations.
- **The FMB problem.** FMB is a proprietary binary. **ASSUMPTION A1:** customers can run Oracle's `frmf2xml` (shipped with Forms builder) to produce Forms XML, or grant the indexer a host where it can be run. The product ingests XML; it treats FMB→XML conversion as a customer-side or agent-side step with a documented recipe. Same for PLL (→ PLD via converter) and RDF (→ XML via rwconverter). This assumption is called out because if false for a given customer, the ingestion adapter must fall back to partial extraction, which degrades but does not break the product.
- **Read-only is not automatic.** "Read-only runtime verification" against a production Oracle DB requires an engineered guarantee (dedicated read-only account, statement allowlisting, resource limits), not a checkbox.
- **Jira write-back is desirable but risky.** Posting RCA summaries back to Jira is a natural closing step but must be gated behind explicit human action.

### 2.4 Explicitly challenged assumptions

- *"The AI should search the source code."* Challenged. The AI should search a **derived, structured index** of the source. Raw-text search over PL/SQL misses semantics (synonyms, package resolution) and floods context. Deterministic parsing first, AI reasoning second.
- *"One pipeline: ticket in, RCA out."* Challenged. A fully automatic pipeline optimizes for demo appeal, not trust. The proposed workflow is **checkpointed**: the engineer can inspect, correct, and steer at three defined gates (extraction review, scoping review, RCA review). Automation level is a per-customer dial.
- *"Graph problems need a graph database."* Challenged in §31/§59: at this scale, recursive CTEs over a properly indexed relational edge table outperform the operational cost of Neo4j on-premise.
- *"Enterprise ⇒ microservices."* Challenged in §29/§59.

---

## 3. Product Vision

**Vision statement:** *Every PREMIA support ticket arrives at the engineer with its investigation already scaffolded — the relevant screens, code paths, and database objects located, the facts separated from the guesses, and the whole trail auditable — so that a two-year engineer with Tracer resolves what previously required a ten-year SME, and the SME's time is spent validating, not excavating.*

What the product is: an **investigation workbench** with an AI investigator inside it. What the product is not: an auto-resolver, a code-fix generator, or a chatbot over the codebase. (Chat exists, but as a scoped assistant inside an investigation, always grounded in the evidence ledger.)

North-star metric: **median time-to-validated-RCA**. Supporting metrics: SME hours per ticket, first-pass artifact-retrieval accuracy, unsupported-claim rate (target: \~0 in validated RCAs).

Product principles:

1. Evidence before eloquence — no claim without a citation the user can click.
2. Human gates at consequential steps — AI drafts, humans decide.
3. The customer's data never leaves the customer — structurally, not contractually.
4. Deterministic where possible, probabilistic only where necessary.
5. Every investigation makes the product measurably better (golden data flywheel).

---

## 4. Target Users

The role model is deliberately smaller than the ten personas listed in the brief. **DESIGN DECISION:** collapse to six product roles with capability flags, because fine-grained role proliferation in enterprise tools produces administration burden without security benefit. L1/L2/Developer/DBA differ in *skill*, not in what the product should let them do; they share the "Investigator" role with optional capability grants.

| Product role | Maps to brief's users | Core rights |
| --- | --- | --- |
| **Viewer** | Viewer, auditors, managers | Read investigations, dashboards. No source view unless granted. |
| **Investigator** | Support Engineer, L1, L2, Developer, DBA | Run investigations, explore code/DB index, draft RCAs. Capability flags: `runtime.request`, `source.view`, `db.explore`. |
| **Validator (SME)** | PREMIA SME | Everything Investigator has + approve/reject RCAs, edit knowledge base, approve runtime probe requests, promote golden records. |
| **Customer Admin** | Customer Administrator | Manage users/roles within their customer, environments, connections, secrets references, indexing schedules, automation-level dial. No RCA content requirement — content access is a separate grant. |
| **Evaluation User** | QA/Evaluation | Golden dataset curation, run eval suites, view metrics. Read-only elsewhere. |
| **Platform Admin** | Platform Administrator | Deployment health, upgrades, licensing, global audit. **Cannot read investigation content by default** (break-glass with audit for support cases). |

**DESIGN DECISION — admin/content separation:** administrators of the system are not automatically readers of investigation content or source code. Insurance customers routinely require this separation; retrofitting it is painful, so it is in the permission model from day one.

---

## 5. User Personas

**Priya — L2 Support Engineer (primary persona).** 3 years on the account, comfortable with SQL, shaky on Forms internals. Receives 5–10 tickets/day; 2 of them require code investigation she can't complete alone. Success for Priya: she opens the ticket in Tracer, confirms the extracted identifiers, watches the investigation land on a form trigger and a package, reads the two cited code snippets, agrees with the confirmed chain, and escalates only the one hypothesis the system marked unconfirmed. Her fear: being handed a confident hallucination and burning a day on it — hence the evidence ledger is *her* feature.

**Rafael — PREMIA SME / Validator.** 14 years of PREMIA. Currently interrupted hourly. Success for Rafael: his queue shows RCAs pre-assembled with clickable evidence; validation takes 10 minutes instead of a 2-hour excavation; his corrections (rejecting a finding, annotating an error code) are captured into the knowledge base so the same mistake doesn't recur. His fear: the tool becoming a source of authority that outranks him — hence validation verdicts are the terminal authority in the data model and in the evaluation ground truth.

**Anita — Customer Admin at the insurer.** Owns the DEV/TEST/UAT/PROD environment definitions, the read-only Oracle accounts, the Jira service credentials, and the security review. Success: onboarding is a guided checklist she can hand to her DBA and Jira admin, secrets live in *her* vault, and she can produce an audit export for her CISO showing exactly which queries ever touched PROD. Her fear: the product phoning home with source code — hence the deployment model and the egress-audit page exist for her.

**Marek — Platform/Delivery engineer at the vendor.** Operates 15 customer cells he mostly cannot look inside. Success: health telemetry (opt-in, content-free), one-command upgrades, and eval-suite results per customer that tell him whether an upgrade regressed retrieval quality *before* the customer notices.

Secondary personas — a Viewer (support manager tracking backlog and MTTR) and an Evaluation user (curates golden tickets, watches unsupported-claim rate across releases) — are served by the dashboard and evaluation surfaces respectively.

---

## 6. Core User Journeys

### Journey 1 — Investigate a ticket (the spine of the product)

1. Priya opens Tracer; the **Ticket Inbox** shows Jira tickets synced from configured projects. She opens INS-4821 ("Settlement fails with ORA-20144 for co-insurance claims").
2. The **Ticket Workspace** already shows: parsed description, comment timeline, two screenshots with OCR overlays, and an **Extraction panel**: error code ORA-20144, claim number pattern, form name `CLMSETTLE` (recognized from a screenshot's window title), business concept "co-insurance".
3. **Gate 1 — extraction review** (skippable per automation dial): Priya corrects one misread identifier, confirms.
4. She clicks **Start Investigation**. A progress rail shows deterministic stages (candidate artifact retrieval → dependency expansion) then the agentic stage, streaming which tools the investigator is calling ("Reading trigger WHEN-VALIDATE-ITEM on SETTLE_DTL… tracing call to PKG_CLM_SETTLE…"). Long-running; she can leave and be notified.
5. **Gate 2 — scoping review** (optional, on by default for new customers): the system presents the candidate artifact set ("this form, these 3 packages, these 5 tables — investigate?") before spending deep-analysis budget. Priya can pin or remove artifacts.
6. The **RCA Workspace** opens: causal chain visualization, findings list with evidence-class badges, source snippets with exact line citations, DB objects involved, one Unresolved item ("could not confirm value of `COINS_APPLICABLE` flag for this claim — runtime probe available").
7. Priya requests the runtime probe (a templated read-only query against UAT). It requires no approval in UAT per policy; result attaches as Runtime-confirmed evidence and flips the hypothesis to confirmed.
8. She edits the draft resolution note and clicks **Submit for validation**.
9. Rafael reviews in the **Validation queue**, opens each cited snippet, confirms two findings, rejects a third with a note ("this branch is dead in v11 — see patch 11.2.4"), and validates the RCA. His rejection auto-creates a knowledge-base suggestion.
10. Priya posts the sanitized summary back to Jira with one click (explicit action, previewed). The investigation is sealed as a historical record and flagged as a golden-candidate.

### Journey 2 — Onboard an environment (Anita)

Guided checklist: create environment → register Oracle read-only account (vault reference) → connectivity test → point indexer at Forms-XML/PLL/RDF paths (or install the site agent that runs conversions) → run first index → review index health report (parse coverage %, unresolved references) → connect Jira project → dry-run extraction on 5 sample tickets → go live. Target: one working day with DBA and Jira admin available.

### Journey 3 — Explore without a ticket (Developer/DBA)

Direct entry into Form Explorer / DB Explorer for ad-hoc questions ("what writes to `POL_PREMIUM_DTL`?"), no investigation object needed. This journey is why explorers are standalone surfaces, not just RCA sub-views.

### Journey 4 — Evaluate a release (Evaluation user + Marek)

Select golden set → replay pipeline stages against pinned snapshots → compare metrics vs. baseline → publish report → gate the upgrade.

---

## 7. End-to-End Workflow

The brief's linear workflow is redesigned into a **checkpointed pipeline with two loops**:

```
            ┌────────────────────────────────────────────────────┐
            │                 KNOWLEDGE LOOP (continuous)         │
            │  Artifact ingestion → parsing → CKG build →        │
            │  incremental re-index → snapshot versions          │
            └────────────────────────────────────────────────────┘
                                   │ (queried by)
 Jira sync → Ticket Workspace → Extraction ──[Gate 1: human confirm*]──▶
   Candidate Retrieval (deterministic, hybrid) ──[Gate 2: scope confirm*]──▶
   Agentic Investigation (LLM + CKG tools, bounded budget) ──▶
   Claim Verification (deterministic) ──▶ RCA Draft ──▶
   [optional Runtime Probes — approval-gated] ──▶
   Engineer Review ──[Gate 3]──▶ SME Validation ──▶
   Seal + Jira write-back* + Golden-candidate flag
            └──────── validation feedback → knowledge base / eval set ────┘
```

\* = configurable per customer/environment (the **automation dial**: Assisted / Checkpointed / Autonomous-draft).

Differences from the brief's workflow, and why:

- **Indexing is decoupled from investigation.** The brief's flow implies analysis happens per-ticket. Parsing a large estate per ticket is unaffordable; the CKG is built continuously and investigations query it. Per-ticket work touches only artifacts not yet indexed at the required depth.
- **Two human gates before the expensive step, one after.** Cheap human confirmation early (10 seconds to fix a misextracted claim number) saves large amounts of wasted deep analysis and is the main lever on RCA accuracy.
- **Claim verification is a separate deterministic stage**, not part of "the AI step". It runs after every agentic output, unconditionally.
- **Runtime probes are pull, not push.** The investigator may *propose* probes; execution requires a human request (and, in protected environments, a second-person approval).
- **Sealing.** Validation seals the investigation immutably (append-only afterwards), which is what makes it usable for audit and as golden data.

---

## 8. Product Information Architecture

Top-level information objects and their containment:

```
Customer (= deployment cell)
 └── Environment (DEV/TEST/UAT/PROD…)            ← isolation + config unit
      ├── Connections (Oracle, Jira, artifact sources, LLM endpoint)
      ├── Index Snapshots (versioned CKG builds)
      │    └── Artifacts → Symbols → Edges → Source spans
      ├── Knowledge Base (base PREMIA KB version + customer overlay)
      ├── Tickets (synced Jira mirror + extraction results)
      ├── Investigations
      │    ├── Evidence Ledger (typed evidence records)
      │    ├── Findings / Claims (cite evidence)
      │    ├── Hypotheses / Unresolved items
      │    ├── Runtime Probe requests + results
      │    └── Review & Validation records
      ├── Golden Records / Eval Runs
      └── Audit Log (append-only)
```

Key IA rules: an Investigation is pinned to exactly one Environment and one Index Snapshot. Tickets can spawn multiple investigations (re-investigation after re-index). Evidence records are immutable; findings reference them, never copy them. The Knowledge Base is layered: vendor-shipped base (per PREMIA version) + customer overlay; overlays win.

## 9. Application Navigation

Left global rail (role-filtered):

1. **Home / My Queue** — assigned tickets, investigations awaiting my action, validation queue (SMEs).
2. **Tickets** — inbox, filters, sync status.
3. **Investigations** — list, states (Extracting, Scoping, Investigating, Draft, In Review, Validated, Sealed, Failed).
4. **Explore** — Form Explorer, Source Explorer, Dependency Explorer, Database Explorer (tabs of one exploration surface sharing a context stack).
5. **Knowledge** — error-code registry, domain concepts, playbooks, customer overlay editor (SME+).
6. **Evaluation** — golden set, eval runs, metric trends (Eval role).
7. **Admin** — environments, connections, indexing, users, automation dial, audit (Customer Admin).

A persistent **environment switcher** in the header scopes everything; PROD is visually distinct (red-tinted chrome) to prevent "wrong environment" errors. A global omnisearch (⌘K) searches tickets, artifacts, symbols, DB objects, knowledge — scoped to the active environment.

## 10. Screen Inventory

| # | Screen | Primary role |
| --- | --- | --- |
| 1 | Home / My Queue | All |
| 2 | Ticket Inbox | Investigator |
| 3 | Ticket Workspace (detail + extraction) | Investigator |
| 4 | Investigation Progress (live pipeline view) | Investigator |
| 5 | Scoping Review (Gate 2) | Investigator |
| 6 | RCA Workspace | Investigator, Validator |
| 7 | Evidence Detail (drawer/full) | All with grant |
| 8 | Runtime Probe Console (request/approve/result) | Investigator, Validator |
| 9 | Validation Queue + Validation view | Validator |
| 10 | Form Explorer | Investigator+ |
| 11 | Source Explorer (code viewer) | Investigator+ (`source.view`) |
| 12 | Dependency Explorer (graph) | Investigator+ |
| 13 | Database Explorer | Investigator+ (`db.explore`) |
| 14 | Knowledge Base (browse/edit/suggestions) | SME |
| 15 | Golden Dataset Manager | Eval |
| 16 | Eval Run + Metrics Dashboard | Eval, Platform Admin |
| 17 | Onboarding Wizard | Customer Admin |
| 18 | Environment & Connections Admin | Customer Admin |
| 19 | Indexing Console (snapshots, health, coverage) | Customer Admin |
| 20 | Users & Roles | Customer Admin |
| 21 | Audit Explorer / export | Customer Admin, Platform Admin |
| 22 | Ops Dashboard (jobs, queues, LLM usage, health) | Platform Admin |

## 11. Detailed UX for Major Screens

**Ticket Workspace.** Three-pane layout. Left: ticket source of truth — description, comment timeline, history, attachments; screenshots open with an OCR overlay where recognized regions (window titles, error dialogs, field labels) are highlighted and clickable. Center: **Extraction panel** — two columns, *Technical identifiers* (error codes, form/block/item/trigger/package names, policy/claim/reference numbers) and *Business context* (symptom summary, operation attempted, product line). Every extracted item shows provenance (which sentence/screenshot region it came from — click to highlight) and a confidence chip; items resolved against the CKG get a green "exists in index" tick, unresolved ones an amber "not found in index" flag (a powerful early signal: a form name that doesn't exist in this customer's estate usually means wrong environment or misread screenshot). Users edit/add/remove chips. Right: actions + related tickets (same error code / same artifacts historically). Primary CTA: *Start Investigation* with automation-dial summary.

**Investigation Progress.** A vertical stage rail (Extraction ✓ → Retrieval ✓ → Scoping ▸ → Deep analysis → Verification → Draft) with, for the agentic stage, a **live tool-call log** in plain language. This transparency is deliberate UX: watching the investigator "read trigger X, follow call to Y" builds calibrated trust and teaches junior engineers the estate. Cancellable; resumable.

**RCA Workspace — the flagship screen.** Header: ticket link, environment + snapshot badge, state, confidence summary ("3 confirmed findings · 1 hypothesis · 1 unresolved"). Body, three panes:

- *Left — Causal chain*: vertical chain Symptom → Entry point (form/item/trigger) → Propagation (calls) → Root cause → Affected data, each node an artifact chip; clicking scopes the other panes.
- *Center — Findings*: ordered claim cards. Each card: claim text, **evidence-class badge** (color-coded: Observed=blue, Derived=teal, Runtime-confirmed=green, Hypothesis=amber, Unresolved=grey), inline citations rendered as chips `[PKG_CLM_SETTLE.VALIDATE_LIMITS:212-238]` that expand the exact source span in place. A card with zero verifiable citations *cannot* display as anything but Hypothesis — the renderer enforces it. Buttons: agree / dispute / annotate.
- *Right — Evidence Ledger*: filterable table of all evidence records (type, source, snapshot, hash, added-by human/agent), plus Unresolved items each with a suggested next step (often a proposed runtime probe), plus the runtime evidence section. Footer: draft resolution editor, Submit for validation, and (post-validation) *Post summary to Jira* with mandatory preview.

**Validation view (SME).** Same RCA workspace plus a verdict bar per finding (Confirm / Reject / Amend, note required on reject), an overall verdict (Validated / Returned / Rejected), a diff view if the RCA was amended after engineer review, and a "promote to golden" toggle. Rejections prompt an optional structured knowledge suggestion ("error ORA-20144 relates to co-insurance limit validation, not reinsurance") that lands in the KB suggestion queue.

**Explorers** — see §14–§17.

**Onboarding Wizard** — see §21.

## 12. Jira Workspace (integration design)

- **Sync model:** pull-based incremental sync (JQL on `updated >=` watermark) per configured project; webhooks are a **FUTURE OPTION** (many enterprise Jira instances won't open inbound webhooks to an internal tool anyway; polling every 1–5 min is sufficient and firewall-friendly). Tracer keeps a **read-mostly mirror** (issue, comments, changelog, attachment binaries) so investigations remain stable even if Jira is slow/unavailable and so extraction runs on local data.
- **Attachment handling:** binaries stored in the cell's object store; images queued for OCR; PDFs/logs text-extracted; size limits and type allowlist per admin config.
- **Write-back:** exactly two write operations, both explicit-human-triggered — post RCA summary comment; transition/link issue (optional, off by default). No silent writes ever.
- **Adapter boundary:** the rest of the product consumes a normalized `Ticket` model (see §46), so ServiceNow or Azure DevOps connectors are additive.
- **Related tickets:** computed locally over the mirror (shared error codes, shared artifacts from past investigations, text similarity) — not via Jira's own linking, which is unreliable.

## 13. Ticket Intelligence (extraction architecture)

**DESIGN DECISION — deterministic-first, LLM-second, human-gated:**

1. **Deterministic extractors** run first: regex/dictionary passes for ORA-/FRM-/REP- codes, PREMIA error-code registry lookups, policy/claim/reference number patterns (per-customer configurable regexes — numbering schemes differ by insurer), and — critically — **dictionary matching against the customer's own CKG symbol table** (form, block, item, trigger, package, table names). Matching against the actual estate rather than generic NER is both more precise and inherently customer-specific.
2. **OCR** (local engine, e.g., Tesseract-class; **FUTURE OPTION:** VLM captioning where a local vision model is approved) over screenshots; recognized text re-enters the deterministic extractors; window-title heuristics map to form titles via the CKG.
3. **LLM pass** for what patterns can't do: symptom summarization, business-concept identification, disambiguation ("the settlement screen" → candidate forms), and reading tables out of pasted logs. The LLM here *proposes*; every proposal is tagged `Derived` with the source sentence, and CKG-resolution decides whether it's plausible.
4. **Gate 1** shows the merged result for human confirmation (skippable on the dial).

Extraction output is a typed `TicketSignals` record — the sole input to retrieval — making stage 1 independently evaluable (see §55).

## 14. Form Explorer

Tree navigation: Form → Blocks → Items / Triggers / Program Units / Record Groups / LOVs / Canvases, backed by the parsed Forms XML. Selecting a node shows: properties (from XML), source (for triggers/program units) with syntax highlighting, **outbound references** (calls, table bindings via block base-table, LOV queries) and **inbound references** ("called by / used by"). A "Where is this item validated?" affordance lists all triggers whose scope touches the item — the single most common SME question. Every node has a stable artifact URL (deep-linkable from RCA citations) and a "add to investigation" action when opened in investigation context.

## 15. Source Explorer

A read-only code viewer over all indexed sources (form triggers/program units, PLL program units, package specs/bodies, procedures, functions, views, report queries) with: symbol-aware search (exact symbol, then lexical, then optional semantic), go-to-definition and find-references powered by the CKG edge table, snapshot switcher (view code as of any index snapshot — essential when investigating old tickets), and span-level permalinks (`artifact@snapshot:lines`) which are exactly what evidence citations resolve to. **DESIGN DECISION:** source is rendered from the indexed copy in the cell's store, never live from customer file shares — investigations must be reproducible against pinned snapshots.

## 16. Dependency Explorer

An interactive graph view seeded from any artifact or from an investigation's artifact set. Edges are typed and filterable: `contains` (form→block→item), `fires` (item→trigger), `calls` (trigger→PU→procedure→package), `reads`/`writes` (code→table/view/column where statically resolvable), `depends-on` (package→package, view→table). Controls: depth limiter (default 2), direction (upstream/downstream), edge-type toggles, and "path finder" (shortest typed paths between two nodes — e.g., form item → table). Every rendered edge carries provenance ("resolved from line 212 of X") — the graph is a *view of evidence*, not a drawing. Rendering caps at a few hundred nodes with clustering beyond that; the full graph is never shown (it's useless).

## 17. Database Explorer

Backed by ingested Oracle catalog metadata (ALL_TABLES/ALL_TAB_COLUMNS/ALL_CONSTRAINTS/ALL_DEPENDENCIES/ALL_SOURCE etc. — see §34), *not* by live connections during browsing. Surfaces: table/view detail (columns, keys, FKs in/out, triggers), package browser (spec/body source, procedures/functions), **cross-boundary references** ("which forms/report queries touch this table" — the join between code-side and DB-side halves of the CKG), and dependency views from ALL_DEPENDENCIES. Live data access happens *only* through the Runtime Probe Console (§19/§36), never from the explorer.

## 18. RCA Workspace

Covered as the flagship screen in §11; its data model in §27. One addition: **investigation chat** — a side panel where the engineer can ask questions ("why do you think the flag is stale?", "show other callers of VALIDATE_LIMITS"). The chat agent has the same CKG toolset and the same claim-verification pass; its answers can be pinned as evidence/annotations. This gives the "chatbot over the codebase" experience safely, inside the evidence regime.

## 19. Runtime Evidence (UX)

The Runtime Probe Console lists **probe templates** (see §36): parameterized, read-only, pre-approved query shapes such as "fetch claim header by claim_no", "show current value + last-updated of column X for key Y", "explain plan for query Q", "count rows matching predicate". An investigator (with `runtime.request`) picks a template — often pre-filled because the investigator agent proposed it against an Unresolved item — binds parameters, selects environment, and submits. Policy per environment decides: auto-run (DEV/TEST), or approval required (UAT/PROD → lands in Validator/Admin approval queue with full rendered SQL shown). Results render as tabular evidence, are hashed, timestamped, tagged `Runtime-confirmed`, and attach to the investigation. Every probe — proposed, approved, executed, denied — is in the audit log with the exact SQL text.

## 20. SME Review

Workflow states: `Draft → In Review (engineer) → Awaiting Validation → Validated | Returned | Rejected → Sealed`. The Validator's verdicts are per-finding and overall; a Returned RCA goes back to the engineer with required notes; validation seals the record (immutable core + append-only comments). Validated content feeds three downstream flows automatically: (1) golden-candidate queue, (2) knowledge-suggestion queue from rejections/amendments, (3) related-ticket index. **DESIGN DECISION:** SME validation is modeled as *ground truth* in the data model — the eval framework (§54–55) treats validated RCAs as labels, which is what makes the flywheel real.

---

## 21. Customer Onboarding

A wizard-driven, resumable checklist owned by the Customer Admin:

1. **Cell provisioning** (vendor-assisted): deploy the Tracer cell (containers) inside the customer network; license key; TLS; SSO/OIDC to the customer IdP.
2. **Environment definition:** create DEV/TEST/UAT/PROD entries; per-environment policy defaults (probe approval rules, automation dial).
3. **Oracle connections:** register TNS details + a **read-only** account per environment (privilege checklist provided: SELECT on catalog views + application schemas; the wizard runs a privilege self-test and *refuses* accounts holding DML/DDL privileges).
4. **Artifact sources:** file-share paths or the **site agent** (a small watcher that runs frmf2xml/rwconverter where the Forms tooling lives and ships XML to the cell). Per-path artifact-type mapping.
5. **First index:** run, then review the **Index Health Report** — parse success rate, artifact counts by type, unresolved-reference count, symbol-collision report. Onboarding does not proceed past a configurable coverage threshold without explicit acknowledgment (a half-indexed estate silently degrades RCA quality; make it visible).
6. **Jira connection:** base URL, service account/PAT (vault reference), project mapping, custom-field mapping (customers put policy numbers in custom fields), identifier regex pack (policy/claim number formats).
7. **Knowledge base:** select shipped PREMIA base-KB version; optionally import customer error-code registry (CSV/SQL) into the overlay.
8. **Dry run:** extraction + retrieval (no deep analysis) over 5–10 sample tickets; admin and an SME eyeball results; tune regexes/mappings.
9. **Users & go-live.**

Per-customer configuration surface (the answer to "what is configurable"): connections & secrets references, paths & converters, identifier regex packs, Jira field mappings, terminology synonyms, KB overlay, automation dial, probe policies, retention windows, LLM endpoint & model choice, indexing schedule, RBAC assignments.

## 22. Multi-Customer Architecture

**DESIGN DECISION (cornerstone): one deployment cell per customer.** Each customer gets a complete, self-contained instance — app, workers, PostgreSQL, object store, optional local LLM — running in *their* network (their VMs, their private cloud account, or a vendor-managed dedicated enclave if they choose). There is no shared application database across customers.

A thin optional **vendor control plane** handles: license issuance, signed release distribution, opt-in *content-free* telemetry (versions, job throughput, error rates, eval metric aggregates — never ticket text, code, or identifiers), and support tooling. Cells function fully with the control plane unreachable (air-gapped mode: licenses and updates by file transfer).

**TRADE-OFF:** N deployments to operate vs. one SaaS. Accepted because (a) the security requirement ("source may not leave the environment") makes shared-infrastructure SaaS a non-starter for most insurers regardless of what we build; (b) cell isolation reduces the isolation-bug blast radius to zero across customers; (c) operational burden is contained by strict homogeneity — every cell runs the identical versioned compose/chart, config as data, upgrades scripted and eval-gated. **FUTURE OPTION:** a shared-control-plane multi-tenant SaaS edition for customers who accept it — the code already keys everything by customer/environment IDs (see §24), so this is a deployment-mode change, not a rewrite.

## 23. Multi-Environment Architecture

Environments are rows, not deployments: one cell hosts all of a customer's environments. Everything downstream is environment-scoped — connections, index snapshots, tickets (a Jira project maps to a default environment, overridable per ticket since a PROD ticket may need investigation against the UAT index), investigations, KB overlay entries (env-specific overrides allowed, e.g., a customization only in PROD), probe policies, and audit partitions. The environment switcher + PROD-tinted chrome carry this into the UX. Cross-environment *diffing* ("this trigger differs between UAT and PROD snapshots") is a **FUTURE OPTION** with high SME value.

## 24. Tenant Isolation

Defense in depth, four layers:

1. **Physical:** per-customer cells (no shared compute/storage/DB across customers).
2. **Schema discipline inside a cell:** every tenant-scoped table carries `environment_id`; PostgreSQL **row-level security** policies enforce environment scoping at the database layer, so an application bug cannot leak across environments; all queries go through a repository layer that requires an environment context object (no context, no query — compile-time enforced).
3. **Job & storage scoping:** every async job, object-store key prefix, and search index is environment-prefixed; workers validate the prefix against the job's context before touching anything.
4. **LLM boundary:** prompts are assembled only from evidence records already scoped by layers 1–3; the LLM gateway tags every call with customer/environment for audit and enforces the per-customer endpoint (a cell configured for local inference physically cannot call an external API — egress-blocked at the network policy level, verifiable on the admin egress page).

Isolation tests are part of CI: a test suite that attempts cross-environment reads through every API surface and asserts zero rows.

## 25. PREMIA Knowledge Architecture

Three knowledge strata, deliberately separated because they have different owners, trust levels, and lifecycles:

| Stratum | Content | Owner | Trust |
| --- | --- | --- | --- |
| **Estate facts (CKG)** | Parsed artifacts, symbols, edges, DB catalog | Machine (deterministic) | Ground truth for "what the code is" |
| **Base PREMIA KB** | Error-code registry, module map (which forms belong to Claims vs Policy vs Endorsement), common failure patterns/playbooks ("ORA-201xx family = application-raised validation errors; check RAISE_APPLICATION_ERROR sites"), PREMIA-version notes | Vendor, versioned, shipped with releases | Curated; every entry cites its basis |
| **Customer overlay** | Customer error codes, customization notes, terminology synonyms ("marine desk" → product code MRN), SME annotations, validated-RCA-derived patterns | Customer SMEs (edit + suggestion queue) | Authoritative for that customer; overrides base |

KB entries are structured records (type, keys, text, links to artifact types, citations), retrievable by the investigator agent through a KB tool. **DESIGN DECISION:** LLM-visible knowledge must be *retrieved and cited*, never baked into prompts as unattributed lore — a KB entry used in reasoning appears in the evidence ledger as `Derived (knowledge)` with its entry ID, keeping the "validated PREMIA domain knowledge" requirement inside the provenance regime. Suggestion pipeline: SME rejections/annotations → suggestion queue → SME approval → overlay entry (versioned, revertible).

---

## 26. AI/LLM Architecture

**Shape: a tool-using investigator agent over deterministic services — not RAG-into-one-prompt, not fine-tuning (initially).**

Components:

- **LLM Gateway** (single choke point): provider adapters (local vLLM/Ollama-served open-weights model; Anthropic/OpenAI/Azure endpoints where the customer permits), per-customer model config, token accounting, rate limits, prompt/response logging (into the cell's own store), redaction hooks, and hard egress rules. Everything AI goes through it; nothing else may call an LLM.
- **Task prompts, small and separated:** extraction assistant, scoping assistant, investigator agent, RCA composer, chat assistant, (later) eval judge. Small single-purpose prompts are testable and swappable; one mega-agent is neither.
- **Investigator agent:** runs a bounded loop (budget: max tool calls, max tokens, max wall time — all per-customer configurable) over a fixed toolset: `search_symbols`, `get_artifact_source(span)`, `get_dependencies(node, edge_types, depth)`, `find_paths(a, b)`, `get_db_object`, `find_error_raise_sites(code)`, `search_knowledge`, `list_ticket_evidence`, `propose_probe(template, params)`, `record_finding(claim, evidence_ids, class)`, `record_unresolved(question, suggested_probe)`. Every tool returns **evidence records with IDs**, and `record_finding` *requires* evidence IDs — the agent literally cannot state a sourceless finding into the ledger. Tool results are truncated/summarized deterministically to keep context bounded (§ Performance).
- **Model strategy:** capability tiers — extraction/summarization can run on a smaller local model; investigation reasoning benefits from the strongest model the customer's policy allows. **ASSUMPTION A2:** some customers will mandate fully local inference; the product must be *useful* (not merely functional) on a strong open-weights model served locally — this is a product-viability requirement, so eval suites run per-model.
- **What LLMs are *not* used for:** parsing, dependency edges, claim verification, RBAC, probe execution, audit — see §28.

**Why agentic rather than single-shot RAG:** the investigation is multi-hop by nature (symptom → form → trigger → package → table). Single-shot retrieval cannot know hop 3 before reading hop 2. A bounded agent that *navigates the CKG* mirrors how SMEs actually work, and the tool-call log doubles as an explainable trace. **TRADE-OFF:** agent loops cost more tokens and latency than one prompt; bounded budgets, deterministic pre-retrieval (which usually seeds the agent within 1–2 hops of the answer), and Gate 2 scoping contain this.

## 27. Evidence / Provenance Architecture

The core schema (simplified):

```
EvidenceRecord {
  id, environment_id, investigation_id,
  class: OBSERVED | DERIVED | HYPOTHESIS | UNRESOLVED | RUNTIME_CONFIRMED,
  kind:  ticket_text | screenshot_region | source_span | ckg_edge |
         db_metadata | kb_entry | probe_result | human_note,
  ref:   { artifact_id, snapshot_id, span } | { probe_id } | { kb_id } | …,
  content_hash, created_by (human|agent|system), created_at, immutable=true
}
Finding { id, claim_text, class, evidence_ids[], status(open/confirmed/rejected),
          verdicts[] (per-reviewer), order_in_chain }
```

Class semantics (enforced, not stylistic):

- **Observed** — exists verbatim in a trusted source: ticket text, screenshot OCR region, indexed source span, catalog metadata. Machine-checkable.
- **Derived** — produced by a deterministic transformation of observed evidence (a resolved edge, a computed path, a KB match). Carries the derivation.
- **Runtime-confirmed** — result of an executed probe. Highest class for state-of-the-world claims.
- **Hypothesis** — any claim whose citations don't fully entail it, or agent inference beyond evidence. Never promoted except by new evidence or SME confirmation.
- **Unresolved** — an explicit open question, with suggested next step.

**Claim Verifier (deterministic stage):** for every finding, (1) every cited evidence ID exists, is in-scope, and its content hash matches; (2) cited source spans still exist in the pinned snapshot; (3) entailment heuristics — cited spans must contain the symbols/error codes the claim names (symbol-level check, deterministic); (4) class consistency — a finding claiming runtime state must cite a probe result or be downgraded. Failures downgrade the finding to Hypothesis and flag it; the verifier's report is itself stored. **TRADE-OFF:** symbol-level entailment is shallow — it catches fabricated citations and misattributed spans, not subtle logical leaps; those remain for the human gates, which is exactly what the gates are for. **FUTURE OPTION:** a second-model entailment check as an additional (still advisory) verifier.

Provenance is end-to-end: snapshot pinning + immutable evidence + append-after-seal gives full reproducibility ("show me exactly what the system saw when it said this in March").

## 28. Deterministic vs AI Responsibilities

| Concern | Deterministic | LLM |
| --- | --- | --- |
| Parsing Forms XML / PLL / PL/SQL / RDF | ✅ | ✗ |
| Symbol & edge extraction, CFG construction | ✅ | ✗ |
| Identifier extraction (codes, numbers, known names) | ✅ (regex/dictionary/CKG match) | assist on ambiguity |
| OCR | ✅ engine | optional VLM assist |
| Symptom summarization, business concepts |  | ✅ (tagged Derived) |
| Candidate retrieval | ✅ hybrid ranker | ✅ query expansion only |
| Investigation navigation (which hop next) |  | ✅ via tools |
| Facts the investigation uses | ✅ tools only return indexed truth | ✗ may not assert |
| Causal narrative composition |  | ✅ over verified findings |
| Claim verification | ✅ | (future advisory 2nd model) |
| Probe SQL | ✅ templates only | may propose template+params only |
| RBAC, isolation, audit, sealing | ✅ | ✗ never |

Rule of thumb applied: **LLMs choose and compose; deterministic code knows and enforces.**

---

## 29. Backend Architecture

**DESIGN DECISION: a modular monolith + worker pool, in one repository, deployed as two container roles (api/ui, worker) sharing PostgreSQL and an object store.**

Internal modules (enforced boundaries — separate packages with explicit public interfaces; an architecture test forbids cross-module imports outside interfaces):

```
core/            identities, environments, RBAC, audit, config
ticketing/       ticket mirror, sync, normalized Ticket model, connectors (Jira…)
ingestion/       source acquisition, converters, parser pipelines (per artifact type)
ckg/             graph schema, symbol table, edge queries, snapshots, path finder
extraction/      ticket-signal extractors (regex/dict/OCR/LLM-assist)
retrieval/       hybrid search & ranking over CKG + KB + ticket history
investigation/   pipeline orchestration, agent runtime, budgets, claim verifier
evidence/        ledger, findings, review/validation workflow, sealing
runtime/         probe templates, approval workflow, probe executor (isolated)
knowledge/       base KB + overlay + suggestion queue
evaluation/      golden records, replay harness, metrics
llm/             gateway, provider adapters, prompt registry, accounting
jobs/            queue, scheduling, retries (Postgres-backed)
```

Why not microservices: the workload is bursty-batch (indexing, investigations) + low-QPS interactive; there is no independent-scaling or independent-team pressure that pays microservices' tax (network failure modes, distributed tracing, N deploy pipelines) — and every customer's ops team would inherit that tax on-premise. The two genuinely separable concerns already are separated: *compute* (workers scale horizontally by count) and *probe execution* (runs in a separately confined runtime module/process because it holds DB credentials — the one place process isolation buys real security). **FUTURE OPTION:** the ingestion/parsing worker can be split out first if parse loads ever demand it; module boundaries make that a packaging change.

API: REST + SSE (job progress, agent tool-log streaming). AuthN via OIDC to customer IdP; AuthZ in core.

## 30. Frontend Architecture

Single-page app (React/TypeScript), one app with route-level code-splitting per surface. State: server-cache library (TanStack-style) + SSE subscriptions for live progress; investigation views are snapshot-addressed so they render identically later. Design system with the evidence-class badge, artifact chip, source-span viewer, and graph canvas as core shared components (they appear on 8+ screens — consistency here *is* the trust UX). Graph rendering via a canvas/WebGL graph lib with node caps + clustering. Code viewer: Monaco-class read-only editor with span-anchor decorations. Accessibility and keyboard-first operation (support engineers live on keyboards; ⌘K omnisearch, j/k queue navigation).

## 31. Data Architecture

**PostgreSQL is the system of record for everything structured**, per environment-scoped schemas:

- Relational core: users/roles, environments, connections, tickets mirror, investigations, findings, evidence, reviews, KB, golden records, audit (append-only table + hash chain), jobs.
- **CKG in Postgres:** `artifact(id, env, snapshot, type, path, hash, meta jsonb)`, `symbol(id, artifact, kind, name, canonical_name)`, `edge(id, env, snapshot, src_symbol, dst_symbol, type, provenance_span)`. Traversals via recursive CTEs with depth caps; hot paths (k-hop neighborhoods of frequently investigated artifacts) optionally materialized.
- Full-text: Postgres FTS/trigram over source, tickets, KB. **Vector embeddings: pgvector, optional per customer** — used only as one ranker among several (see §33); the product must work without it (some customers won't run embedding models).
- **Object store** (S3-compatible, e.g., MinIO in-cell): raw artifact files, converted XML, attachments, screenshots, probe result payloads, eval reports — content-addressed (hash) for dedup and integrity.

**TRADE-OFF (no Neo4j):** at 10⁴–10⁵ artifacts and roughly 10⁵–10⁶ edges per snapshot, recursive CTEs with proper indexes answer depth-≤4 typed traversals in milliseconds; a second database engine on-premise doubles the ops burden and splits transactional consistency (evidence must reference edges transactionally). Revisit only if graph analytics (centrality, similarity at scale) become product features — **FUTURE OPTION** behind the `ckg` module interface.

## 32. Storage Strategy

- **Snapshots, not mutation:** each index run produces a snapshot; unchanged artifacts (by content hash) are *shared* between snapshots (copy-on-write rows), so storage grows with churn, not with snapshot count. Retention policy per environment (e.g., keep all snapshots referenced by open/sealed investigations + last N).
- Attachments/probes: content-addressed blobs, retention per customer policy, legal-hold flag on sealed investigations.
- Backups: standard pg_dump/WAL + object-store replication, runbook shipped with the cell; **the customer owns backups** (their data, their keys).
- Encryption at rest (disk or pgcrypto for secrets-adjacent columns) and TLS everywhere in-cell.

## 33. Search / Retrieval Architecture

Candidate retrieval (pipeline stage between extraction and agent) is a **deterministic hybrid ranker** — this stage decides most of the product's accuracy, so it is evaluable and tunable, not agentic:

1. **Exact signals first:** extracted error codes → `find_error_raise_sites` (every RAISE_APPLICATION_ERROR / error-registry match in the CKG); extracted symbol names → direct CKG hits; form recognized from screenshot → that form's subtree.
2. **Structural expansion:** 1–2 hop typed neighborhoods of exact hits (a trigger's called packages; a table's writing procedures).
3. **Lexical:** FTS over source + KB with customer synonym packs.
4. **Historical:** artifacts implicated in validated RCAs of similar past tickets (similarity over signals, not embeddings-only).
5. **Semantic (optional):** pgvector similarity as an additional ranker. Scores fuse with per-signal weights (defaults shipped; tuned per customer against their golden set — this is what the eval harness optimizes). Output: a ranked, capped candidate set (default ≤ 25 artifacts) → Gate 2 → agent seed.

**DESIGN DECISION:** retrieval is *recall-oriented and cheap*; the agent is *precision-oriented and expensive*. Keeping them separate keeps both measurable (form-retrieval accuracy is literally stage-3 recall@k on golden tickets).

## 34. Artifact Processing Architecture

Per-artifact-type parser adapters behind one `ArtifactParser` interface (`accepts(type) → parse(blob) → {symbols, edges, spans, diagnostics}`):

- **Forms:** FMB → XML via frmf2xml (site agent or customer-run recipe); XML parsed to form/block/item/trigger/PU structure; trigger & PU bodies extracted as PL/SQL units and passed to the PL/SQL front end.
- **PL/SQL:** grammar-based parser (ANTLR-class PL/SQL grammar) → AST → symbol defs/refs, call edges, static table/column references (SELECT/INSERT/UPDATE/DELETE targets, `%TYPE`/`%ROWTYPE`), RAISE_APPLICATION_ERROR sites with code literals, and per-unit **control-flow graphs** (see §35). Parse failures degrade gracefully: unparseable units still get lexical indexing + a diagnostics flag (never silently drop code — the health report shows coverage).
- **PLL:** → PLD/XML conversion, then PL/SQL front end; library-attachment edges to forms.
- **Reports:** RDF → XML; extract data-model queries (→ table references), triggers, and layout field bindings.
- **Database:** catalog ingestion over the read-only connection — objects, columns, constraints/FKs, ALL_DEPENDENCIES (server-side dependency truth), and ALL_SOURCE for stored PL/SQL (parsed by the same front end). **DESIGN DECISION:** ALL_DEPENDENCIES edges and parser-derived edges are stored with distinct provenance and cross-checked; disagreements surface in the index health report (they usually indicate synonyms/dblinks or parse gaps).
- **Name resolution:** a resolution pass links references to definitions using Oracle scoping rules (schema, synonyms table, package scope); unresolved references are kept as explicit `unresolved_ref` nodes (they are evidence of *something* — often the customization boundary).

Adding an artifact type = adding one adapter (§46).

## 35. Dependency / CFG Architecture

- **Dependency layer:** the typed edge table (§31) with provenance spans; query API: neighborhoods, typed shortest paths, reachability with depth caps, inbound/outbound reference lists. This is the substrate for the Dependency Explorer, agent tools, and the claim verifier.
- **Control-flow layer:** per PL/SQL unit, a CFG (basic blocks, branches, exception handlers, RAISE edges) computed at parse time and stored compactly. Primary product uses: (a) **error-path tracing** — from a RAISE/exception site, walk backward to the conditions guarding it ("this error fires when `v_coins_flag = 'Y' AND v_limit < v_amount`") — the single highest-value automated analysis in this domain; (b) rendering the path in the RCA chain; (c) feasibility hints ("this branch requires X"). **Scope discipline:** intra-procedural CFG + call-graph stitching only; whole-program dataflow/symbolic execution is deliberately out (undecidable-in-practice on a 10⁵-unit estate, and runtime probes answer the "what is the actual value" question far more cheaply). **FUTURE OPTION:** targeted inter-procedural constant/flag propagation for the specific "which config flag guards this branch" question.

## 36. Runtime Evidence Architecture

- **Probe templates** are code-reviewed, versioned, parameterized SQL shapes shipped with the product (+ customer-authored templates approved by Customer Admin + Validator). Parameters are strongly typed and bound (no string concatenation, ever).
- **Executor:** a separate confined process holding the per-environment read-only credentials; enforces: allowlisted statement shape (parse-checked: single SELECT/EXPLAIN, no DML/DDL/PLSQL-block), row/byte caps, statement timeout, Oracle resource-limited profile on the account, session tagging (`MODULE=TRACER_PROBE`) for the customer's own DBA visibility.
- **Policy engine per environment:** auto-run / single approval / dual approval; approver sees fully-rendered SQL + business justification (linked Unresolved item).
- **Guarantee stack for "read-only":** account privileges (no DML grants) → statement allowlist parse → template review → audit of exact text + result hash. Four independent layers; any one failing still leaves three.
- The agent can only *propose* (template_id, params) — proposals are data, not executions.

## 37. Jira Integration Architecture

The `ticketing/` module defines the normalized model (`Ticket`, `Comment`, `Attachment`, `ChangeEvent`, `TicketRef`) and a `TicketingConnector` port (`fetchUpdatedSince`, `fetchAttachment`, `postComment`, `capabilities()`). The Jira adapter implements it for Jira Cloud + Data Center (auth: PAT/OAuth per instance), handles rate limits with adaptive backoff, custom-field mapping declared in config, and idempotent sync via changelog cursors. Failure isolation: Jira downtime pauses sync; investigations proceed on the mirror. ServiceNow/AzureDevOps = new adapters only. Write scope of the service account should be commented-on-issues only — the wizard verifies and warns on over-privileged accounts (same philosophy as the DB account check).

---

## 38. Security Architecture

Threat model headliners and their answers:

| Threat | Control |
| --- | --- |
| Source/data exfiltration to vendor or third parties | Per-customer cell in customer network; egress allowlist (default: none beyond configured Jira/Oracle/LLM endpoints); admin **Egress page** showing every configured external endpoint; air-gap mode |
| Cross-environment leakage | RLS + context-required repositories + scoped storage keys + CI isolation tests (§24) |
| Prompt injection via ticket content/attachments | Ticket text is *data* in prompts, never instructions; tool allowlist is fixed; agent cannot execute probes or writes; claim verifier catches fabricated citations; write-backs human-gated |
| Credential theft | Secrets never in app DB (references only, §39); probe executor process isolation; short-lived session tokens |
| Malicious/compromised insider (vendor) | Vendor has no standing access to cells; break-glass is customer-granted, time-boxed, fully audited |
| Over-privileged integration accounts | Wizard privilege self-tests refuse/warn (DB and Jira) |
| Tampering with investigation record | Immutable evidence, sealed investigations, hash-chained audit log |

Plus the standard enterprise baseline: OIDC SSO + MFA via customer IdP, TLS 1.2+ everywhere, dependency scanning + signed releases (supply chain), session policies, CSP-hardened frontend, attachment type/size allowlists and malware scanning hook.

## 39. Secrets Management

- **Principle: the product stores references, not secrets.** First-class integration with the customer's vault (HashiCorp Vault, CyberArk, AWS/Azure/GCP secret managers) via a `SecretProvider` port; connections store `vault://path` references resolved at use-time with short-lived leases.
- Fallback for customers without a vault: an in-cell encrypted secret store (envelope encryption; master key from KMS/HSM or sealed keyfile under customer control) — clearly labeled as the lesser option in onboarding.
- Scope of secrets: Oracle accounts (per environment), Jira credentials, LLM API keys, SSO client secrets, site-agent tokens. Rotation: re-resolve on lease expiry; connection health checks surface rotation breakage immediately.
- Secrets never appear in logs, prompts, evidence, or exports (redaction filters at the logging and LLM-gateway layers, tested).

## 40. RBAC

Model: **role + capability flags + environment scope grants.** A user holds roles (§4) per environment set (e.g., Investigator in DEV/TEST/UAT, Viewer in PROD). Sensitive capabilities are explicit grants layered on roles: `source.view`, `db.explore`, `runtime.request`, `runtime.approve`, `kb.edit`, `golden.manage`, `audit.read`, `jira.writeback`. Enforcement: single policy module in `core/` consulted by API layer *and* repository layer (defense in depth); UI hides what the API would deny (never the only line of defense). Admin/content separation per §4. All permission changes are audited events. **DESIGN DECISION:** no custom per-object ACLs in v1 (investigations visible to all Investigators of that environment) — per-investigation confidentiality is a **FUTURE OPTION**; premature object ACLs are the classic enterprise-tool complexity trap.

## 41. Auditability

Append-only, hash-chained `audit_event` stream per environment: who/what/when/context for — logins & permission changes; connection & secret-reference changes; index runs & snapshot creation; investigation lifecycle transitions; every LLM call (prompt hash, model, tokens — full text retained in-cell per retention policy); every probe (proposed/approved/denied/executed + exact SQL + result hash); Jira write-backs (full posted text); KB changes; exports; break-glass access. Audit Explorer UI + filtered CSV/JSON export + optional syslog/SIEM forwarder (customers will ask for Splunk/QRadar feeds). Sealed investigations + pinned snapshots + immutable evidence give *investigative* auditability: any past RCA is fully reconstructible.

## 42. Observability

In-cell, self-contained (no SaaS dependency): structured logs (correlation IDs across API→job→agent→tool call), metrics (Prometheus format: queue depths, job latencies by stage, parse coverage, LLM tokens/latency/cost per stage, probe latencies, sync lag), traces for the pipeline (each investigation is a trace; each tool call a span — this triples as the user-visible tool log), and an Ops Dashboard for Platform/Customer admins. Health checks per dependency (DB, object store, Jira, Oracle, LLM endpoint) drive the connection-status UI. Opt-in content-free telemetry to the vendor control plane (§22). Alert rules shipped: sync stalled, index failure rate, verifier downgrade spike (a *quality* alarm — a jump in downgraded claims means model or prompt drift), queue saturation.

## 43. Scalability

Envelope targets (per cell): 10⁵ artifacts, 10⁶–10⁷ edges across retained snapshots, 10⁵ historical tickets, 50 concurrent users, 20 concurrent investigations, full re-index of a large estate overnight and incremental re-index in minutes.

How the shape scales: interactive traffic is trivial (low QPS, read-heavy, indexed); the real load is **batch** — parsing and agent runs — which scales by adding worker containers (stateless, queue-fed; parse jobs partition by artifact, agent jobs by investigation). PostgreSQL sizing at this envelope is a single well-provisioned instance (+streaming replica for HA); the CKG edge table at 10⁷ rows with composite indexes is unremarkable. LLM throughput is the true bottleneck under concurrency → the gateway queues with per-stage priorities (interactive chat > investigation > eval batch) and the ops dashboard makes LLM saturation visible so the customer can size inference hardware. Scaling *customers* = adding cells (linear, no shared bottleneck). What deliberately does **not** scale in v1: cross-customer anything.

## 44. Modularity

Module map in §29; the rules that keep it honest: each module owns its tables (no cross-module SQL); communication through interfaces + domain events on the job bus (e.g., `SnapshotCompleted`, `InvestigationValidated` — consumed by evaluation and knowledge modules without coupling); ports at every external boundary (§46); an architecture-conformance test suite (import rules, table ownership) in CI. The practical payoff enumerated by the brief: swapping Jira→ServiceNow, LLM provider, DB engine target (Oracle→ others for non-PREMIA estates — **FUTURE OPTION**), artifact types, PREMIA versions (a KB + parser-profile concern, not a code fork), knowledge sources, and probe providers are all single-adapter changes.

## 45. SOLID Principles

Applied where they earn their keep, concretely:

- **S:** parser ≠ indexer ≠ retriever ≠ agent ≠ verifier ≠ composer — each stage one reason to change (a new PL/SQL construct touches the parser only; a new evidence class touches evidence + verifier only).
- **O:** new artifact types, connectors, LLM providers, probe templates, extractors, and rankers are *additive registrations* against stable interfaces — no core edits.
- **L:** connector/provider contracts specified with capability descriptors (`capabilities()`) so a lesser implementation degrades declared features instead of violating expectations (e.g., a ticketing adapter without changelogs falls back to full-diff sync).
- **I:** narrow ports per concern — the agent's toolset interface is separate from the explorer query interface even where they share implementation, so agent capabilities are governable independently.
- **D:** `investigation/` (the RCA engine) depends only on ports: `TicketSource`, `GraphQuery`, `SourceStore`, `Knowledge`, `LLM`, `ProbeProposer`, `EvidenceSink`. It compiles and tests with all of them faked — which is precisely the brief's "don't couple the RCA engine to external systems," made mechanical.

## 46. Plugin / Adapter Strategy

Extension points (all in-process plugins — versioned interfaces + registry; out-of-process plugin sandboxing is a **FUTURE OPTION**, unneeded while all plugins are vendor/customer-admin-installed):

| Port | Ships with | Later |
| --- | --- | --- |
| `TicketingConnector` | Jira Cloud + DC | ServiceNow, ADO, email intake |
| `ArtifactParser` | FormsXML, PL/SQL, PLL, RDF-XML, OracleCatalog | Custom config formats, APEX |
| `LLMProvider` | vLLM/OpenAI-compat local, Anthropic, Azure OpenAI | others |
| `SecretProvider` | Vault, AWS/Azure/GCP, in-cell store | CyberArk |
| `ProbeExecutor` | Oracle read-only SQL | AWR/ASH reader, log fetcher, REST health probes |
| `KnowledgeSource` | Base KB + overlay | Confluence importer, doc ingestion |
| `Extractor` | regex/dict/OCR/LLM extractors | customer-specific extractors |
| `Ranker` | exact/structural/lexical/history/vector | learned ranker |

Config-as-data per customer selects and parameterizes plugins; the eval harness runs per plugin combination so substitutions are measurable, not vibes.

## 47. Async Processing

Everything that can exceed \~2 s is a job: ticket sync, OCR, extraction, indexing (per-artifact fan-out), retrieval+agent investigations, probe execution, eval runs, exports. **DESIGN DECISION: PostgreSQL-backed queue** (`SELECT … FOR UPDATE SKIP LOCKED`) rather than Kafka/RabbitMQ — at hundreds-to-thousands of jobs/hour, Postgres queues are proven, transactional with the domain data (job + state change commit atomically — a real correctness win), and zero extra ops for on-premise customers. Job model: priorities, per-type concurrency limits, retries with backoff + dead-letter state surfaced in ops UI, idempotency keys, cancellation (agent loops check a cancel flag between tool calls), progress events over SSE. **TRADE-OFF:** a Postgres queue tops out far below Kafka — irrelevant at this envelope; the `jobs/` port allows a broker swap if an envelope-breaking use ever appears.

## 48. Incremental Processing

- **Indexing:** content-hash diffing per artifact file → only changed artifacts re-parse; edges re-resolved for changed units + their direct dependents (bounded ripple); snapshots share unchanged rows (§32). Site agent watches paths and triggers deltas; scheduled full-verify pass weekly catches drift.
- **DB catalog:** `LAST_DDL_TIME`-driven delta pulls.
- **Ticket sync:** cursor-based (§37). Extraction re-runs only on changed tickets.
- **Retrieval indexes** (FTS/vector): updated per changed artifact, not rebuilt.
- **Investigations are never silently re-run** on new snapshots — staleness is *shown* ("snapshot 42 investigated; current is 45 — re-investigate?"), preserving reproducibility while inviting refresh.

## 49. Versioning

Everything user-meaningful is versioned: index snapshots (the backbone); base KB (semver, shipped per release, per-PREMIA-version profiles); customer overlay entries (row-versioned, revertible); prompts and agent policies (prompt registry with versions — an RCA records which prompt/model versions produced it, enabling honest before/after eval); probe templates; plugins and their interface versions; the product itself (single-artifact releases, DB migrations forward-compatible one version for rollback). An investigation is thus a fully pinned tuple: (snapshot, KB versions, prompt versions, model, plugin set) — the precondition for both audit and evaluation.

---

## 50. Deployment Architecture

A cell = one docker-compose (or Helm chart for customers who insist on their K8s) with services: `app` (API+UI), `worker` (×N), `probe-executor` (confined), `postgres`, `minio` (object store), optional `inference` (vLLM) and `site-agent` (deployed separately near the Forms tooling). Sizing profiles shipped: S (evaluation, 1 VM), M (typical, 2–3 VMs, \~64 GB + optional GPU box for local inference), L (large estate/users). Upgrades: pull signed images (or load from file in air-gap), run migrations, run the customer's eval smoke set, flip. Vendor control plane (optional, outbound-only from cell): license, update channel, content-free telemetry.

## 51. On-Premise Deployment

On-premise is the *primary* mode, not a port: no hard external dependencies (queue=Postgres, store=MinIO, metrics=Prometheus-local, LLM=local option), air-gap support (file-based license/updates/KB imports), customer-controlled TLS/PKI and IdP, resource-capped defaults, and a printed-quality runbook (backup/restore, DR, health triage) because the operator is the customer's IT, not the vendor's SREs. The design constraint that most shaped the architecture — "assume the operator is a competent enterprise IT team with no interest in this product" — is why the moving-part count is minimized throughout.

## 52. Failure Handling

Failure domains and behavior:

- **Jira unreachable:** sync pauses with visible lag banner; everything else proceeds on the mirror.
- **Oracle unreachable:** catalog refresh + probes degrade; investigations proceed on last catalog snapshot with a staleness badge on DB evidence.
- **LLM endpoint down/slow:** gateway retries/fails over per config; investigations park in `Waiting (LLM)` state, resumable — never lost; deterministic surfaces (explorers, search) unaffected — the product degrades to a *very good code navigator*, which is deliberate.
- **Parser failure on an artifact:** per-artifact diagnostic, lexical-only fallback, coverage report — never aborts the snapshot.
- **Agent failure/budget exhaustion:** partial findings + Unresolved items are still delivered with an explicit "investigation incomplete" banner (partial evidence is valuable; silent failure is not).
- **Worker crash mid-job:** lease timeout → retry from checkpoint (agent checkpoints after each tool call).
- **Verifier downgrades everything:** the RCA is presented as hypotheses-only with a quality warning — the honest outcome.
- **Probe timeout/row-cap:** result marked truncated; never auto-retried against PROD.

## 53. Test Strategy

- **Parser tests:** golden-file corpus per artifact type (real anonymized FMB-XML/PLL/RDF samples + synthetic PL/SQL grammar coverage); differential tests vs. ALL_DEPENDENCIES for edge extraction.
- **CKG/query tests:** property-based traversal tests (path symmetry, depth caps), snapshot-sharing correctness.
- **Isolation tests:** the cross-environment attack suite (§24) — release-blocking.
- **Verifier tests:** adversarial fixtures — fabricated citations, hash mismatches, class violations — must all downgrade.
- **Agent tests:** recorded-LLM (cassette) tests for tool-loop mechanics; budget/cancel/checkpoint behavior; plus live-model eval (below) as the semantic test.
- **Contract tests** per port (every `TicketingConnector` etc. runs one shared conformance suite).
- **E2E:** dockerized synthetic estate ("mini-PREMIA": \~50 forms, \~200 packages, seeded bugs) + mocked Jira → full pipeline through validation.
- **Security tests:** RBAC matrix sweep, probe allowlist fuzzing (DML/DDL/injection attempts must be rejected at parse), secret-redaction tests.
- **Performance tests:** index throughput on synthetic 10⁵-artifact estates; traversal latency SLOs.

## 54. Golden Dataset

A golden record = a **sealed, SME-validated investigation** promoted by an Eval user, containing: the frozen ticket (text, comments, attachments), pinned snapshot reference, expected artifacts (form/block/item/trigger/packages/db objects — extracted from validated findings, SME-editable), expected failure path, validated RCA text, resolution, and validation metadata. Sources: (1) the natural flywheel — every validated investigation is a candidate (one click to promote, PII-review step included); (2) backfill mode during onboarding — SMEs replay historical solved tickets through the pipeline and label results, which doubles as customer acceptance testing; (3) the synthetic mini-PREMIA set shipped with the product for vendor-side regression. Golden records are per-customer and never leave the cell; the vendor sees only aggregate metrics (opt-in). Storage: first-class tables in `evaluation/`, snapshot-pinned so replays are exact.

## 55. Evaluation Metrics

Stage-wise, because the pipeline is stage-wise (each stage independently replayable against golden records):

| Stage | Metrics |
| --- | --- |
| Extraction | identifier precision/recall by type (error code, claim no., form name…); OCR field accuracy |
| Retrieval | recall@k and MRR per artifact class — *form / item / trigger / package / db-object retrieval accuracy* map here |
| Investigation | correct-failure-path rate; findings precision vs. expected artifacts; tool-call efficiency (calls per confirmed finding) |
| Verification/faithfulness | **unsupported-claim rate** (findings downgraded by verifier ÷ findings emitted) — the headline safety metric; citation-validity rate |
| RCA | RCA correctness (SME-judged match to known RCA: exact / partially / wrong); hypothesis calibration (fraction of hypotheses later confirmed) |
| Runtime | probe-confirmation accuracy (probe outcomes vs. expectations on goldens where applicable) |
| End-to-end | latency P50/P95 per stage and total; cost (tokens) per investigation; human-effort proxy (edits per RCA, validation minutes) |

Operational use: eval runs execute per release candidate per customer cell (their goldens, their model, their config) and gate upgrades; trend dashboards expose drift; ranker weights (§33) are tuned against retrieval metrics. **FUTURE OPTION:** an LLM judge for RCA-match scoring to reduce SME judging load — always advisory, never ground truth.

---

## 56. MVP Definition

Scope discipline: the MVP must complete Journey 1 end-to-end for one design-partner customer, with the evidence regime fully present (it is the identity of the product, not a v2 feature).

**In (MVP):** per-customer cell deployment (compose); multiple environments per customer; Jira DC/Cloud connector (read + comment write-back); ingestion of Forms-XML + PL/SQL (packages/procedures/functions/triggers/PU) + Oracle catalog; CKG + snapshots + incremental hashing; deterministic extraction + OCR + LLM assist; hybrid retrieval (exact/structural/lexical/history — vector excluded); investigator agent with the fixed toolset + budgets; claim verifier; RCA workspace + evidence ledger + validation workflow + sealing; Form/Source/Dependency/DB explorers (read); RBAC (6 roles + flags), audit log, vault-reference secrets; onboarding wizard; golden-record promotion + stage-wise eval harness + mini-PREMIA regression set; ops dashboard.

**Out (explicitly):** runtime probes (stubbed as "suggested manual query" text in MVP — the approval/executor machinery follows in the first post-MVP release), PLL/RDF parsers (lexical-only fallback in MVP), vector search, investigation chat, cross-snapshot diffing, ServiceNow, SaaS mode, learned ranker, VLM screenshot understanding.

**MVP success criteria (design partner):** ≥80% form-retrieval recall@10 and ≥70% package/db-object recall@10 on a 30-ticket backfilled golden set; unsupported-claim rate \<5% pre-validation and 0 in validated output; median assisted investigation time ≤ 50% of the customer's historical baseline on comparable tickets; SME validation time ≤ 15 min median.

## 57. Post-MVP Roadmap

- **R2 (hardening + runtime):** probe templates + executor + approval policies; PLL + RDF parsers; investigation chat; SIEM forwarding; Helm packaging.
- **R3 (retrieval + knowledge):** optional pgvector ranker; KB suggestion pipeline UX; cross-snapshot diff explorer; related-ticket surfacing v2; learned ranker weights from goldens.
- **R4 (breadth):** ServiceNow connector; multi-model tiering in gateway; eval LLM-judge (advisory); backfill tooling at scale; reporting/dashboard pack for managers.
- **R5 (depth):** targeted inter-procedural flag propagation (§35); AWR/ASH probe provider; environment-diff-aware RCA ("works in UAT, fails in PROD" comparative mode).

## 58. Future Extensions

Fix-suggestion drafts (patch *proposals* with the same evidence regime — never auto-applied); proactive mode (watch incoming Jira, pre-run extraction+retrieval so tickets arrive pre-scaffolded); cross-customer *pattern* KB (vendor-curated, abstracted, opt-in contribution with review — never raw customer data); coverage analytics ("which modules generate most tickets"); non-PREMIA Oracle-Forms estates (the architecture is Forms-generic; PREMIA specificity lives in the KB); SaaS edition (§22); APEX/ADF artifact parsers for customers mid-migration; training mode (junior engineers replay sealed investigations as guided walkthroughs — the tool-call log becomes a teaching artifact).

## 59. Architecture Trade-offs (consolidated)

1. **Per-customer cells vs. multi-tenant SaaS.** Chose cells. Pay: N deployments, slower fleet-wide iteration, harder usage insight. Get: structural data isolation (the requirement that dominates), sellability to insurers, zero cross-tenant blast radius. Mitigation: homogeneous automated cells + content-free telemetry. Reversible: yes (SaaS as a future mode), because tenancy keys exist throughout.
2. **Modular monolith vs. microservices.** Chose monolith + workers. Pay: single-runtime blast radius, discipline needed to keep boundaries. Get: on-prem operability by non-experts, transactional simplicity (evidence+jobs+audit in one DB), 1 pipeline. Mitigation: enforced module boundaries + the one real isolation need (probe executor) split out. Reversible: per-module, later.
3. **Postgres-as-graph vs. Neo4j.** Chose Postgres. Pay: no native graph algorithms; recursive CTEs need care. Get: one database to operate/back-up/secure, transactional evidence↔edge integrity. Reversible behind `ckg/` port.
4. **Postgres job queue vs. broker.** Chose Postgres. Pay: throughput ceiling. Get: atomic job+state commits, zero extra ops. Reversible behind `jobs/` port.
5. **Agentic investigation vs. single-shot RAG.** Chose bounded agent. Pay: tokens, latency, loop-engineering. Get: multi-hop correctness, explainable trace, natural probe proposals. Mitigation: deterministic pre-retrieval seeds it; budgets cap it.
6. **Deterministic verifier vs. trusting the model.** Chose verifier. Pay: engineering + shallow-entailment limits. Get: mechanical enforcement of the no-invention requirement; the headline trust metric becomes measurable.
7. **Checkpointed workflow vs. full automation.** Chose human gates with a dial. Pay: added interaction cost on easy tickets. Get: trust, accuracy, and training-data quality; the dial lets mature customers earn automation.
8. **Local-first LLM support vs. best-API-only.** Chose both via gateway, with local as a first-class citizen. Pay: quality variance across models, per-model eval burden. Get: the deals where data cannot leave; honest per-model metrics.
9. **Index-time analysis vs. query-time analysis.** Chose index-time (CKG) with query-time depth only in the agent. Pay: storage + freshness machinery (snapshots, incremental). Get: fast investigations, reproducibility, bounded AI context.
10. **Six roles vs. ten.** Chose collapse+flags. Pay: occasional mapping conversations with customers. Get: administrable RBAC that customers actually configure correctly.

## 60. Architecture Decision Records

**ADR-01 — Per-customer deployment cell.** *Status: Proposed.* Context: source/DB/Jira content must not leave customer environments; insurers audit vendors hard. Decision: single-tenant cells in customer networks; optional content-free control plane. Options: multi-tenant SaaS (rejected: unsellable to target segment, isolation-bug risk), hybrid data-plane/control-plane split with vendor-hosted control (kept as the telemetry-only variant). Consequences: fleet ops automation becomes a product deliverable; SaaS remains possible later.

**ADR-02 — Modular monolith + async workers on Postgres queue.** *Status: Proposed.* Context: on-prem operators; batch-heavy load; small vendor team. Decision: one app + worker role; enforced module boundaries; Postgres `SKIP LOCKED` queue. Options: microservices+K8s+Kafka (rejected: ops tax exported to customers, no scaling need), serverless (rejected: on-prem). Consequences: simple DR/runbooks; must police boundaries with CI conformance tests; broker/module extraction paths reserved.

**ADR-03 — Deterministic Code Knowledge Graph in PostgreSQL as sole AI-visible truth.** *Status: Proposed.* Context: no-invention requirement; multi-hop domain; huge estates. Decision: parse-time symbols/edges/CFG with provenance; LLM accesses only via tools; Neo4j rejected (ops cost, transactional split). Consequences: parser quality is the product's foundation (invest accordingly); graph-analytics features would force revisiting.

**ADR-04 — Typed evidence ledger + deterministic claim verifier.** *Status: Proposed.* Context: distinguish Observed/Derived/Hypothesis/Unresolved/Runtime-confirmed; enforce citations. Decision: immutable evidence records; findings require evidence IDs; verifier downgrades unverifiable claims; renderer refuses uncited confidence. Options: prompt-level instructions only (rejected: unenforceable), second-LLM critic as primary control (rejected as primary: unfalsifiable; kept as future advisory). Consequences: some verbose bookkeeping; the product's core differentiator becomes testable.

**ADR-05 — Checkpointed pipeline with per-customer automation dial.** *Status: Proposed.* Context: wrong-RCA cost asymmetry; trust building; golden-data quality. Decision: Gates at extraction, scoping, review/validation; dial Assisted/Checkpointed/Autonomous-draft. Consequences: slightly slower easy tickets; far better hard-ticket accuracy and label quality.

**ADR-06 — Hybrid deterministic retrieval separated from the agent.** *Status: Proposed.* Context: retrieval decides accuracy; must be measurable/tunable per customer. Decision: exact→structural→lexical→history(→optional vector) fused ranker feeding a capped candidate set. Consequences: retrieval metrics map 1:1 to the brief's accuracy metrics; vector infra optional.

**ADR-07 — Probe templates + confined executor for runtime verification.** *Status: Proposed.* Context: read-only guarantee against production Oracle. Decision: reviewed parameterized templates; separate credential-holding process; statement-parse allowlist; per-environment approval policy; agent proposes only. Options: free-form agent SQL behind a read-only account (rejected: single-layer guarantee, injection surface, unauditable intent). Consequences: template library curation becomes ongoing product work; probes are slower but defensible to any CISO.

**ADR-08 — LLM Gateway with local-first provider strategy.** *Status: Proposed.* Context: data-boundary variance across customers; model churn. Decision: single gateway choke point, provider adapters, per-customer model config, egress enforcement, full in-cell logging, per-model eval. Consequences: provider swaps are config; quality variance is surfaced honestly by eval rather than hidden.

**ADR-09 — Snapshot-pinned investigations with incremental indexing.** *Status: Proposed.* Context: continuously changing estates; audit/reproducibility; eval replay. Decision: content-hash incremental snapshots with row sharing; investigations pin (snapshot, KB, prompt, model) tuples; staleness surfaced, never silently refreshed. Consequences: storage/retention machinery; exact reproducibility of every sealed RCA.

**ADR-10 — Validation-driven golden flywheel.** *Status: Proposed.* Context: eventual golden dataset requirement; scarce SME time. Decision: sealed validated investigations promote to golden records; backfill mode doubles as acceptance testing; stage-wise replay harness gates releases per cell. Consequences: evaluation is a lived process, not a project; SME verdicts must be captured with structure from day one (they are).

---

## Closing note on independence

Nothing above assumes any prior POC architecture. Where the brief's implied design was challenged, the challenge and reasoning are recorded (§2.4, §7, §59). The three bets most worth debating when comparing against the alternative proposal are: **cells over SaaS (ADR-01), the deterministic CKG + verifier as the trust mechanism (ADR-03/04), and the checkpointed-agent workflow over full automation (ADR-05)** — these determine the product's character; most other choices are reversible behind ports.