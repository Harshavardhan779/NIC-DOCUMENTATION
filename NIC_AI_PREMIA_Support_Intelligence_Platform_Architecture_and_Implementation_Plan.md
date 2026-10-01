# NIC AI PREMIA Support Intelligence Platform
## Final Architecture & Phased Implementation Plan

**Status:** Architecture baseline / implementation roadmap  
**Method:** Superpowers → Mimo/OpenCode → Tests → gstack → Human acceptance  
**Current phase:** Phase 9 — Deterministic Core Integrity + Identity

---

# 1. Purpose

This document is the master architecture and implementation roadmap for evolving the existing NIC AI RCA POC into the **NIC AI PREMIA Support Intelligence Platform**.

Core principle:

> **Deterministic analysis is the source of truth. LLMs generate hypotheses and reasoning. Runtime evidence confirms behavior where required. SME validation remains the final authority.**

---

# 2. Existing Foundation

The existing POC is the engineering foundation and must be preserved.

Validated capabilities include:

- project dump ingestion
- artifact registry
- incremental FMB conversion
- Oracle Forms FMB → XML conversion
- Forms XML extraction
- dependency extraction
- SQLite canonical indexing
- FTS retrieval
- deterministic CFG analysis
- RCA evidence retrieval
- retrieval-quality metadata
- deterministic sufficiency gate
- bounded RCA context generation
- OpenAI-compatible LLM provider abstraction
- strict RCA response validation
- runtime evidence foundation
- extensive automated regression testing

Current corpus scale is approximately:

- 1,091 indexed form artifacts
- 138k items
- 244k dependencies
- 56k CFG graphs
- 57k triggers
- 7.6k program units
- 8.1k indexed errors

---

# 3. Final Architecture

```text
                         SUPPORT UI
      Ticket · Investigation · RCA · SME Validation · Explorers
                                |
                                v
                         APPLICATION API
                                |
        +-----------------------+------------------------+
        |                       |                        |
        v                       v                        v
 Ticket Intelligence     Investigation Engine        Evaluation
 Jira / OCR /             Sufficiency Gate           Golden Data
 Attachments              Deterministic RCA          Replay
 Untrusted Data                    |
                                   |
                         +---------+---------+
                         |                   |
                         v                   v
                  Deterministic RCA     Gate-Fail Path
                                           |
                                  +--------+--------+
                                  |                 |
                                  v                 v
                           Multi-hop Retrieval    Agent
                                                EXPERIMENT
                                  |                 |
                                  +--------+--------+
                                           |
                                           v
                                  Evidence Ledger
                                           |
                                           v
                              Citation Integrity
                                  Verification
                                           |
                                           v
                                  Runtime Evidence
                                           |
                                           v
                                    SME Validation
                                           |
                                           v
                                    Golden Dataset
```

Underlying architecture:

```text
                    CKG TOOL / QUERY CONTRACT
                              |
                 +------------+------------+
                 |                         |
                 v                         v
          CORPUS STORE              OPERATIONAL STORE
          SQLite snapshots          PostgreSQL
          Immutable                 Transactional
                 |
                 v
          Ingestion Workers
          FMB → XML
          Parsers
          Indexing
```

Runtime:

```text
Runtime Executor
      |
      +-- isolated process
      +-- read-only credentials
      +-- templates only
      +-- human approval
      +-- audit
      +-- result-data governance
```

Deployment:

```text
Customer A
└── Customer Cell
    ├── Environment A
    ├── Environment B
    └── Environment C

Customer B
└── Separate Customer Cell
```

---

# 4. Architectural Principles

## Deterministic Core

The deterministic repository is authoritative for source facts.

## LLM

The LLM may interpret evidence and generate hypotheses, but must not invent source, runtime, database, or execution facts.

## Runtime

Runtime evidence is controlled confirmation, never autonomous execution.

## SME

Final RCA correctness requires static evidence, runtime confirmation where applicable, and SME validation.

## CKG

CKG is a logical graph model and query/tool contract over the existing deterministic index. It is not a second graph datastore.

## Modular Monolith

Start as a modular monolith with explicit interfaces. Extract services only when security, workload, or scaling justifies it.

---

# 5. Storage Architecture

## Corpus Store — SQLite

Use one immutable SQLite file per corpus snapshot.

Contains:

- artifacts
- forms
- blocks
- items
- triggers
- program units
- dependencies
- CFG
- errors
- FTS

SQLite remains the corpus store.

## Operational Store — PostgreSQL

Use PostgreSQL for:

- customers
- environments
- tickets
- investigations
- evidence
- jobs
- audit
- users
- SME validation
- runtime records

Confirm the operational choice with a narrow concurrency benchmark.

## Job Queue

Use PostgreSQL queue semantics such as:

```sql
SELECT ... FOR UPDATE SKIP LOCKED
```

Do not introduce Kafka/RabbitMQ at the current scale.

---

# 6. Identity and Snapshot Model

## Canonical Artifact Identity

Artifact identity must never depend on scan or insertion order.

Target:

```text
artifact identity =
    canonical repository-relative path
    +
    content hash
```

## Derived Identity

```text
derived identity =
    source identity
    +
    toolchain version
```

This applies to XML, extracted JSON, dependencies, CFGs, and indexes.

## Snapshot

A reproducible snapshot consists of:

```text
source content set
+
extraction/parser toolchain versions
+
CFG/analyzer version
+
relevant retrieval configuration
+
LLM model/prompt version for an investigation run
```

---

# 7. CKG Tool Contract

Expose bounded deterministic operations such as:

```text
search_symbols()
find_error_sites()
find_dependencies()
get_form()
get_trigger()
get_program_unit()
get_cfg()
get_callers()
get_callees()
find_related_items()
```

Every tool must provide:

- snapshot awareness
- provenance
- deterministic ordering
- bounded results
- stable schemas

The physical corpus remains authoritative.

---

# 8. Investigation Domain

Core entities:

```text
Customer
Environment
Snapshot
Ticket
Investigation
InvestigationRun
Finding
Claim
EvidenceReference
RuntimeProbe
SMEValidation
```

Relevant entities carry:

```text
customer_id
environment_id
snapshot_id
```

Investigations must be reproducible.

---

# 9. Sufficiency Gate

The existing deterministic sufficiency gate remains.

```text
Ticket
  |
  v
Deterministic Retrieval
  |
  v
Sufficiency Gate
  |
  +---- SUFFICIENT ----> Deterministic / bounded RCA
  |
  +---- INSUFFICIENT --> Gate-fail investigation path
```

The agent is not the default replacement for the gate.

---

# 10. Ticket Intelligence

Inputs:

- Jira ticket
- comments
- history
- screenshots
- attachments
- OCR
- customer descriptions

Extract:

```text
issue_type
business_area
action
failure_indicator
error_codes
identifiers
form_names
item_names
database_objects
procedure_names
technical_terms
timestamps
```

All ticket-derived content is **UNTRUSTED DATA**, including OCR.

---

# 11. Retrieval

Retrieval ladder:

```text
1. Exact identifiers
2. Error codes
3. Structural relationships
4. CFG paths
5. Lexical retrieval
6. Historical RCA
7. Optional vector retrieval
```

Vector retrieval remains deferred until lexical and structural recall is measured.

Historical RCA retrieval must prevent evaluation contamination.

---

# 12. Evidence Ledger

The ledger is append-only and introduced before production agent work.

Evidence contains:

- evidence ID
- snapshot
- artifact identity
- content hash
- source span
- provenance
- evidence class
- supersedes relationship where applicable

Claim classes:

```text
HYPOTHESIS
STATIC_OBSERVED
RUNTIME_CONFIRMED
SME_VALIDATED
```

Claims cannot silently upgrade between classes.

---

# 13. Citation Integrity Verification

The verifier checks:

- citation exists
- evidence exists
- snapshot matches
- hash matches
- source span exists
- symbol exists
- evidence class is compatible
- provenance is valid

It does **not** prove causal correctness.

Use:

```text
Citation Integrity: VERIFIED
```

not:

```text
RCA Correctness: VERIFIED
```

RCA correctness comes from evaluation, runtime confirmation where applicable, and SME validation.

---

# 14. Runtime Evidence

Runtime execution is:

- templates-only
- read-only
- human-approved
- isolated
- audited
- bounded

Flow:

```text
Approved Probe Template
        |
        v
Authoring-time SQL parser validation
        |
        v
Parameter validation
        |
        v
Human approval
        |
        v
Isolated executor
        |
        v
Read-only database
        |
        v
Runtime evidence
```

No free-form SQL reaches runtime.

Runtime results require classification, retention, access control, masking policy, and explicit rules for whether results may enter LLM context.

---

# 15. Bounded Agent

The agent is experiment-gated.

Compare:

```text
A. Existing deterministic pipeline
B. Deterministic multi-hop expansion
C. Bounded tool-using agent
```

Evaluate:

- RCA correctness
- evidence quality
- unsupported claims
- latency
- token cost
- tool calls
- replayability
- failure rate

Productionize only if the experiment demonstrates measurable benefit.

---

# 16. Customer / Cell Architecture

Use one deployment cell per customer.

Each cell contains the customer's:

- corpus
- environments
- operational data
- investigations
- evidence
- runtime configuration
- model policy

Central multi-customer control plane is deferred.

---

# 17. Security

Critical requirements:

- corpus encryption/access control
- controlled snapshot export
- ticket/OCR prompt-injection containment
- isolated runtime executor
- read-only credentials
- templates-only runtime
- human approval
- evidence/audit integrity
- explicit LLM locality and data-egress policy
- PII/result-data governance

The customer source corpus is treated as crown-jewel data.

---

# 18. Evaluation

Golden records contain:

```text
Historical ticket
+
validated investigation
+
expected artifacts
+
known RCA
+
resolution
+
SME validation
```

Evaluation snapshots must freeze source and toolchain versions.

Metrics:

### Retrieval

```text
Form Recall@K
Item Recall@K
Trigger Recall@K
Dependency Recall@K
Error Recall@K
```

### RCA

```text
Root-cause correctness
Contributing-factor correctness
Resolution correctness
```

### Evidence

```text
Citation validity
Citation precision
Unsupported-claim rate
Evidence completeness
```

### Runtime

```text
Runtime confirmation rate
```

### System

```text
Latency
Token usage
Tool calls
Failure rate
SME review time
```

Include negative controls whose correct result is:

```text
INSUFFICIENT
NOT_REPRODUCIBLE
```

---

# 19. Full Implementation Roadmap

## Phase 9 — Integrity + Identity

Goal: make the deterministic core internally consistent.

Tasks:

```text
9.1 Canonical identity specification
9.2 Identity migration / aliasing
9.3 H1 CFG anchor resolution
9.4 H2 analyzer-version derivation
9.5 H3 pipeline/indexer state contract
9.6 H5 provider/CLI error boundary
9.7 H6 templates-only runtime SQL
9.8 Regression corpus freeze
9.9 Full regression
9.10 gstack review
9.11 Phase acceptance
```

Phase 9 must NOT implement:

- Investigation domain
- Evidence Ledger
- PostgreSQL
- CKG adapter
- Jira
- UI
- retrieval expansion
- vector search
- agent

## Phase 10 — Investigation Domain Model

Implement the domain entities and customer/environment/snapshot relationships.

## Phase 10.5 — Thin Vertical Slice

```text
Real ticket
→ extraction
→ deterministic retrieval
→ RCA
→ evidence
→ minimal SME validation
→ golden record
```

Begin accumulating evaluation data.

## Phase 11 — CKG Tool Contract

Implement the logical graph/query contract over the deterministic index.

No second graph store.

## Phase 12A — PLL/PLD Coverage

Improve corpus completeness and scope negative evidence correctly.

## Phase 13 — Ticket Intelligence

Implement Jira ingestion, attachments, OCR, signal extraction, and injection containment.

## Phase 16 — Evidence Ledger + Citation Integrity

Implement append-only evidence, claim classes, provenance, supersedes, and citation verification.

## Phase 14 — Retrieval 2.0

Improve deterministic retrieval and evaluate against the growing golden dataset.

## Phase 15 — Agent Experiment

Compare deterministic, multi-hop deterministic, and bounded-agent approaches.

## Phase 17 — Runtime Integration

Integrate controlled runtime evidence into investigations.

## Phase 18 — Jira Productization

Implement ticket workspace and human-approved Jira write-back.

## Phase 19 — Full UI / Explorers

Implement Ticket, Investigation, RCA, Form, Dependency, and Database workspaces.

## Phase 20 — PREMIA Knowledge

Add PREMIA and insurance-domain knowledge overlays without replacing source truth.

## Phase 21 — Customer / Cell Maturity

Productionize customer-cell architecture and defer centralized control plane.

## Phase 22 — Operational Storage Benchmark

Benchmark PostgreSQL concurrency, queues, investigation writes, and audit workloads.

## Phase 23 — Evaluation Platform Hardening

Formalize frozen datasets, replay, metrics, contamination controls, and negative controls.

## Phase 24 — Production Hardening

Finalize security, observability, audit, backup, recovery, access control, secrets, LLM locality, PII handling, retention, encryption, isolation, performance, and upgrade strategy.

---

# 20. Coding-Agent Operating Model

Every implementation follows:

```text
Superpowers Specification
        ↓
Superpowers Implementation Plan
        ↓
ONE approved task
        ↓
Mimo/OpenCode
        ↓
Targeted tests
        ↓
Regression tests
        ↓
gstack review
        ↓
Fix/retest
        ↓
Human acceptance
        ↓
Next task
```

The coding agent must not:

- redesign architecture
- implement future tasks
- modify unrelated files
- weaken tests
- invent requirements
- silently change interfaces

If it finds an architectural conflict:

```text
STOP
→ report conflict
→ explain impact
→ propose options
→ wait for approval
```

---

# 21. Standard Coding-Agent Prompt

```text
You are implementing Task <TASK_ID> from the approved Superpowers implementation plan.

Read the approved specification and plan before modifying code.

Rules:

1. Implement ONLY this task.
2. Do not redesign the architecture.
3. Do not implement future tasks.
4. Do not modify unrelated files.
5. Do not weaken or delete tests to make them pass.
6. Preserve existing public contracts unless the approved task explicitly changes them.
7. Prefer the smallest correct implementation.
8. Follow existing repository conventions.
9. If the specification conflicts with the existing implementation, STOP and report the conflict instead of improvising.

Before coding:
- inspect the relevant implementation,
- inspect existing tests,
- identify affected interfaces.

After coding:
- run task-specific tests,
- run relevant regression tests,
- report changed files,
- report exact test commands/results,
- report deviations,
- report remaining risks.

Do not proceed to another task without approval.
```

---

# 22. gstack Operating Model

```text
Implementation
    ↓
Tests
    ↓
gstack review
    ↓
Findings
    ↓
Fix
    ↓
Tests
    ↓
gstack re-review
```

gstack is a review layer, not the architecture authority.

Superpowers specifications and approved architecture remain authoritative.

---

# 23. Phase Acceptance Gate

A phase is complete only when:

```text
[ ] Specification satisfied
[ ] Implementation plan satisfied
[ ] Acceptance criteria satisfied
[ ] Targeted tests pass
[ ] Regression suite passes
[ ] gstack review completed
[ ] Critical review findings resolved
[ ] No architecture drift
[ ] No unrelated modifications
[ ] Documentation updated
[ ] Security implications reviewed
[ ] Evidence/artifacts captured
[ ] Human acceptance received
```

---

# 24. Immediate Next Action

Do NOT start Phase 10.

Do NOT implement the product UI.

Do NOT implement the agent.

Do NOT migrate storage.

Start with Phase 9.

Immediate workflow:

```text
Architecture validation COMPLETE
        ↓
Create Phase 9 Superpowers SPEC
        ↓
Review / approve SPEC
        ↓
Create Phase 9 Superpowers PLAN
        ↓
Review / approve PLAN
        ↓
Task 9.1
        ↓
Mimo/OpenCode implementation
        ↓
Tests
        ↓
gstack review
        ↓
Fix/retest
        ↓
Task 9.2
        ↓
...
```

First artifact:

```text
docs/superpowers/specs/
2026-09-27-phase-9-integrity-identity.md
```

The Phase 9 specification must cover:

- canonical identity
- existing-ID migration/alias strategy
- provenance continuity
- H1-H6 remediation boundaries
- regression corpus freeze
- acceptance criteria
- explicit non-goals


