Architecture Validation Report — NIC AI PREMIA Support Intelligence Platform (Converged Architecture)
1. Executive Assessment

The converged architecture is directionally correct: preserving the tested deterministic RCA core, adopting Tracer's product concepts (Investigation, Evidence Ledger, cells, evaluation) as a layer around it, and refusing premature technology migration are all sound instincts. However, the plan as written contains four structural defects that would cost you months if not corrected before Phase 9:

Defect 1 — The Evidence Ledger is sequenced after the components that depend on it. The bounded investigator (Phase 15) has record_finding in its tool list, and findings reference evidence IDs. Claim verification (Phase 16) verifies against the ledger. You cannot introduce the ledger after the agent. Worse, the ledger's core guarantee — immutable evidence — is impossible while H4 (scan-position-dependent artifact IDs) is unfixed, which the plan does acknowledge, but the ledger's identity spec is nowhere scheduled. Identity must be designed in Phase 9, not merely "fixed."

Defect 2 — The golden-dataset flywheel is starved by the phase ordering. Golden records require SME-validated investigations. SME validation requires a validation workflow and at least a minimal UI. The plan puts UI at Phase 19 and evaluation at Phase 23. That means for roughly ten phases, no golden data accumulates, and Retrieval 2.0, the agent, and claim verification all ship without the evaluation harness that is supposed to justify them. This inverts the plan's own stated philosophy ("only migrate based on evidence").

Defect 3 — Claim verification is being sold as something it is not. Decision G's checks (evidence exists, hash matches, span exists, symbols cited) verify citation integrity, not causal validity. A perfectly cited, hash-valid, snapshot-consistent RCA can still be causally wrong. If the product presents "verifier passed" as "RCA verified," you have built a trust-laundering machine. The verifier must be renamed and scoped honestly (see Section 8).

Defect 4 — Storage is treated as one decision when it is two. The corpus index (read-heavy, immutable per snapshot, single-writer at ingestion) and the operational store (investigations, evidence, jobs, audit — concurrent writers, transactional, long retention) have opposite characteristics. Benchmarking "SQLite vs PostgreSQL" as a monolithic migration question will produce a muddled answer. Split the question and most of it answers itself (Section 5).

Verdict: conditionally ready for Phase 9 — with Phase 9 scope amended (identity specification, not just identity fix) and the phase sequence reordered as proposed in Section 23.

2. Architecture Coherence

The layered flow (deterministic retrieval → seed → bounded agent → ledger → verification → RCA → runtime → SME) is coherent as a conceptual pipeline, with three coherence gaps:

Snapshot semantics are underspecified. Decision B says investigations must be reproducible against their historical snapshot. But H2 (analyzer version staleness) demonstrates that a "snapshot" of source content alone is insufficient: derived artifacts (CFGs, dependency edges, indexes) depend on toolchain versions. A reproducible snapshot must be defined as (source content set, extraction toolchain version vector). Two investigations against the same source with different CFG analyzer versions are against different snapshots. This definition must exist before Phase 10's schema, or Investigation reproducibility is a fiction.

The sufficiency gate's role in the converged flow is unstated. The existing engine has a deterministic sufficiency gate before the LLM call. The converged diagram shows retrieval → seed → agent with no gate. Does the gate still fire? If yes, what does the agent do that the gate's failure branch doesn't? If no, you've removed a tested safety mechanism. My recommendation: the gate stays, and the agent is the escalation path when the gate fails (insufficient evidence), not a replacement for the single bounded call when the gate passes. This resolves question F cleanly (Section 7).

Negative evidence coherence depends on corpus completeness, and Phase 12 undermines it. The POC produces negative evidence ("no such trigger exists"). That claim is only as good as the corpus. Phase 12 defers PLL/PLD coverage — but PL/SQL libraries are where a large fraction of real PREMIA logic lives. Until PLL is ingested, every "not found" is really "not found in the subset we parse," and any RCA leaning on negative evidence is unsound. Negative evidence must carry a corpus-scope qualifier, and PLL coverage should move earlier (see Section 16).

3. Existing POC Preservation Assessment — Decision A: KEEP

Preserving the engine is correct and the reasoning is strong: 1,091 forms, 244k dependencies, 56k CFGs, extensive tests — this is falsifiable, benchmarked infrastructure. Rewriting it to match Tracer's shape would discard the only empirically validated asset you have.

Depends on: the six H-findings being genuinely fixable in place (they are — none is a design-invalidating flaw; H1/H3/H4 are identity/consistency bugs, not architectural dead-ends).
What could break: H4's fix. Changing artifact identity retroactively breaks provenance of everything already indexed. The fix needs a migration/aliasing plan, not just new ID generation.
Reversibility: high — preservation is the reversible choice by definition.
Change before implementation: treat H4 as a design task (canonical identity spec, Section 19/T) rather than a bug fix.
4. CKG Assessment — Decision C: MODIFY

Here is the over-abstraction question (Q&A: B, C) answered directly: a canonical graph model is justified; a canonical graph store is not.

Your SQLite index already is a graph in relational clothing: artifacts, items, triggers, program units, dependencies as edges, CFGs. What Tracer's CKG actually buys you is a stable contract — a named set of node/edge/span concepts that the agent tools, the explorers, the verifier, and the evaluation harness all speak — decoupled from physical schema. That contract is valuable and cheap.

What would be over-abstraction: materializing a second "semantic graph" datastore that mirrors the deterministic index. That creates a synchronization problem (which store is authoritative when they disagree?), doubles storage, and directly violates the "deterministic core is the source of truth" principle by introducing a second truth.

Simplest counterexample for the two-store design: analyzer version bump regenerates CFGs in the index; the CKG copy lags; the agent's get_cfg and the verifier's span check now disagree about the same trigger. You've manufactured H2 at a new layer.

So: CKG = the tool/query API contract over the existing index (the Decision E tool list is the CKG interface), plus the logical concept vocabulary from Decision C. No new store. Validation: define the tool contract, implement it as a thin adapter over SQLite, and confirm every explorer/agent/verifier need is expressible through it. If something isn't expressible, that's a real signal — extend the index, not fork it.

5. Storage Assessment — Decisions D(implicit), K: MODIFY; O (PG job queue): ADOPT

Split the storage question (answers to C, D, E, O, P):

Corpus index (per-snapshot, immutable after ingestion, read-heavy): SQLite is not merely adequate here — it is architecturally advantageous. One SQLite file per snapshot gives you: snapshot pinning as a file handle, backup as a file copy, reproducibility as content-hash of the file, WAL-mode concurrent readers for parallel investigations, and zero server operations for on-prem cells. At 138k items / 244k edges / 56k CFGs, everything fits comfortably; recursive traversal at this scale is milliseconds in either engine. Migrating this to PostgreSQL buys you nothing and costs you the snapshot-as-file property. Keep SQLite for the corpus, and stop benchmarking it as a migration candidate — benchmark it only for regression.

Operational store (investigations, evidence ledger, tickets, jobs, audit, users): this has concurrent multi-process writers, cross-entity transactions, row-level security potential, long retention, and audit requirements. SQLite's single-writer model is the wrong shape here once you have an API server plus workers. PostgreSQL for the operational store is the near-certain outcome — but per Decision K's own philosophy, confirm with a narrow benchmark: concurrent investigation writes + job queue under worker contention. That's the only benchmark that matters; the Decision K list (FTS, CFG lookup, traversal) benchmarks the wrong store.

E — Is PostgreSQL a suitable graph substrate at this scale? Yes, trivially. 244k edges with bounded-depth recursive CTEs and cycle guards is small. Neo4j is unjustified — correctly on your avoid-list. The only workload that could ever stress this is unbounded find_paths; bound it (depth, fanout, result count) at the tool contract and the question disappears.

O/P — Job queue: PostgreSQL with SELECT ... FOR UPDATE SKIP LOCKED handles this workload (dozens of concurrent investigations, ingestion jobs, OCR jobs) with an enormous margin. Kafka/RabbitMQ would be justified only by: sustained thousands of jobs/second, multi-consumer fan-out streaming, or cross-cell event replication — none of which exist in a per-customer cell. ADOPT PG queue; REJECT Kafka (your own constraint list already says this; I'm confirming the workload analysis supports it).

C — Can SQLite evolve without becoming tech debt? In the split model, yes — because it stops being asked to do things it's bad at. It becomes tech debt only if you push operational, multi-writer data into it.

6. Retrieval Assessment — Decision D: KEEP, with two amendments

The ladder (exact → error → structural → CFG-path → lexical → historical → optional vector) is sound and correctly ordered from deterministic to probabilistic.

Amendment 1 — historical-RCA retrieval creates evaluation contamination. Once validated RCAs feed retrieval, any golden record whose answer is retrievable-verbatim inflates every metric. You must partition: golden eval records are excluded from the retrieval-visible knowledge store (or evaluated in a mode with historical retrieval disabled, reported separately). Without this, Phase 23's numbers are unfalsifiable.

Amendment 2 — vector retrieval: DEFER, and note the on-prem constraint now. On-prem cells mean local embedding models, local vector index, and per-snapshot re-embedding cost. Don't spend design effort until lexical+structural recall is measured and found wanting on real tickets. The layered design makes this perfectly reversible — the correct reason to defer.

What could break the ladder overall: ticket signals are garbage (OCR noise, user-paraphrased error text). Retrieval quality is upstream-bounded by extraction quality — which is why extraction precision/recall (Decision L) must be measured first, not alongside.

7. Agent Assessment — Decision E: DEFER + EXPERIMENT REQUIRED

Direct answers to F and G:

F — Is a bounded agent genuinely better than the current single bounded RCA call? Unproven, and the burden of proof is on the agent. The single-call design is deterministic-ish, cheap, testable, and currently working. The agent adds: nondeterministic tool trajectories, multiplied token cost, replay difficulty, a larger injection surface, and an evaluation problem (trajectory variance means N runs per golden record). The only principled justification is the case the current design handles worst: the sufficiency gate fails — evidence is insufficient, and iterative exploration might find what one-shot retrieval missed.

Therefore: the experiment is precisely specified. Take real tickets where the sufficiency gate failed or the RCA was wrong for retrieval reasons. Measure whether an agent with the Decision E tools converts them to correct RCAs at acceptable cost, versus (a) the current pipeline and (b) the cheap alternative — a deterministic multi-hop retrieval expansion (one more structural expansion round when the gate fails). If (b) closes most of the gap, the agent is not justified yet. Run this experiment only after the evaluation harness exists — another reason to reorder phases.

G — Where autonomy stops (lock these as invariants now, even though the agent is deferred):

No side effects outside the investigation record: record_finding, record_unresolved, propose_runtime_probe are the only writes, and the probe is proposal-only — execution requires human approval, always.
No Jira write, no filesystem, no free SQL, no shell, no network — already correct in Decision E.
Hard budgets on tool calls, tokens, wall time, recursion — already listed; add per-tool result size caps and deterministic result ordering (stable sorts). Without ordering determinism, replay against a pinned snapshot still diverges, and reproducibility (Decision B) fails for agent-driven investigations.
All tool outputs and ticket content enter the context as data with provenance, never as instructions (injection containment, Section 14).
8. Evidence / Verifier Assessment — Decision F: ADOPT (resequenced); Decision G: MODIFY

H — Does the ledger materially improve trust or add excessive complexity? It materially improves trust, conditional on stable identity. The ledger is what turns "the LLM said X" into "claim X cites evidence E, E has hash h, h resolves in snapshot S." That is the product's core differentiation and it is worth its complexity. But: with H4 unfixed, evidence IDs referencing scan-position-dependent artifact IDs are quicksand — an unchanged corpus rescanned in different order silently orphans the ledger. Ledger schema work must not start until the identity spec (T) is locked. Add: append-only with explicit supersedes links, and class-transition rules (HYPOTHESIS may never be promoted to OBSERVED without a new evidence record; RUNTIME_CONFIRMED only ever originates from the runtime layer).

I — Can deterministic claim verification detect unsupported causal reasoning? No, and the architecture must say so out loud. Decision G's checks establish that citations are real, in-scope, hash-valid, and class-consistent. That is citation integrity. It cannot detect:

J — Claims that inevitably escape symbol-level verification:

Causal direction errors ("A causes B" where both A and B are correctly cited but the arrow is wrong).
Selective citation / omission — the RCA ignores correctly-indexed exculpatory evidence.
Quantified and universal claims ("this trigger always fires before commit") — the CFG shows a path exists, not that it always executes.
Runtime-state inferences from static evidence ("the parameter must have been null") — partially caught by "runtime claims require runtime evidence," but only if claim classification correctly labels the claim as runtime; misclassification is the escape hatch.
Configuration- and data-dependent behavior, version skew between customer environment and ingested snapshot, and cross-artifact interaction effects.
Negative claims that exceed corpus scope (Section 2's PLL problem).

Modifications: (1) rename the verifier's output to something like citation-verified, never "RCA verified"; UI and reports must reflect that RCA correctness is established only by runtime confirmation + SME validation. (2) Structure claims as subject–relation–object with a declared claim class, so the verifier can at least check relation-type against evidence class mechanically. (3) Downgrade-not-reject is right — keep it. (4) The residual gap is exactly what the golden dataset's "RCA correctness" and "unsupported-claim rate" metrics exist to measure — which again argues for evaluation infrastructure earlier.

9. Runtime Security Assessment — Decision H: KEEP, with hardening locked

K — Sufficiently isolated? Architecturally yes (separate layer, fail-closed, templates, approval, audit). Three concrete gaps for PROD (question L):

H6 proves regex/blacklist SQL validation is a dead end. Quoted-identifier bypass is the canary. The fix is not a better regex: lock the templates-only rule — no free-form SQL ever reaches the validator; templates are validated with a real SQL parser at authoring time by a human-review workflow, and runtime validation is reduced to parameter typing/bounds. This converts an unwinnable input-validation problem into a tractable change-management problem.
The executor must be a separate OS process/host with read-only credentials scoped per customer-environment, statement timeouts, row caps, and no network path back except the result channel. In cells this is cheap; do it from the start.
Result data classification is missing entirely from the plan. PREMIA is insurance: runtime probe results can contain policyholder PII, financial data, claims data. The architecture has audit for execution but nothing for result handling — retention, masking, who can view probe output, whether probe results are allowed into LLM context at all. This is the largest missed security item in the runtime layer (feeds Section 14/Y).

Approval policy for PROD: SME approval per probe minimum; consider two-person approval for production databases. NOT_EXECUTED as the default state and no autonomous execution — keep permanently, not just "initially."

10. Jira / Product Assessment — Decision I: ADOPT (phased)

Sound scope. Two additions: (1) ticket content is the primary prompt-injection vector — ticket text, comments, and OCR'd screenshots are attacker-writable (customers, and anyone who can email a screenshot into a ticket). All ticket-derived content must be structurally segregated as untrusted data in every LLM context, and Phase 24's injection testing should specifically test instructions embedded in screenshots surviving OCR. (2) Write-back must be human-approved per action with a rendered preview — lock this; never batch-approve.

OCR quality will bound extraction quality; measure extraction P/R on real attachments before building retrieval features that consume signals (correctly ordered in Phase 13 → 14 — keep that ordering).

11. Multi-Customer Assessment — Decision J: MERGE; cells: ADOPT

M — Are cells justified? Yes, for three reasons that are specific to this product, not generic preference: on-prem deployment is a stated requirement (cells are the on-prem model); insurance data isolation is contractual/regulatory, and deployment isolation is both stronger and vastly simpler to demonstrate to a customer's security team than row-level tenancy; and PREMIA installations are heavily customized per customer, so corpora, overlays, and golden data genuinely don't overlap — shared tenancy would share almost nothing.

N — Multi-customer before single-customer completion? No. But do the cheap thing now: customer_id and environment_id in the Phase 10 schema from day one, and no cross-customer identifiers anywhere. That costs almost nothing and removes the future migration. DEFER the control plane (fleet management, central telemetry, cross-cell upgrades) entirely — it's a real product but not this year's.

12. Modularity / SOLID Assessment — modular monolith: ADOPT

Q — Boundary strength: adequate if module boundaries are enforced as the CKG tool contract + explicit internal APIs, with a lint/CI rule preventing cross-module DB access. The realistic failure mode of modular monoliths is not the pattern — it's schema-level coupling where every module reads every table. The split-store decision (Section 5) helps: corpus SQLite is only reachable through the CKG adapter.

R — Extraction candidates (design their interfaces as if remote, don't extract yet):

FMB→XML conversion — this one is not optional to isolate: it depends on the Oracle Forms toolchain, likely a different OS/runtime than everything else. It is already effectively a separate component; formalize it as one behind the job queue.
Runtime executor — extract for security isolation (Section 9), independent of scale.
OCR/attachment processing — CPU/GPU heavy, spiky; first performance-driven extraction.
LLM gateway — for provider governance and cell-level model policy.
Everything else (retrieval, ledger, verifier, investigation engine) stays in-process.
13. Scalability Assessment

Per-cell scale is modest and the corpus numbers prove it: hundreds of thousands of rows, not billions. Concurrency needs are tens of simultaneous investigations, not thousands. The scaling dimensions that actually matter: (a) ingestion time for full re-index on analyzer version bumps (H2 fix will cause full regenerations — measure and budget this; 56k CFG regenerations must be parallelizable and resumable); (b) LLM throughput/cost per investigation, especially if the agent lands; (c) OCR throughput on attachment backlogs. None requires distributed systems. The avoid-list (K8s, Kafka, microservices) survives scrutiny for the per-cell workload; a single beefy VM or two per cell is the honest sizing.

14. Security Assessment

Y — Largest missed risks, in order:

Runtime probe result data governance (Section 9.3) — PII in probe outputs, unaddressed.
Prompt injection via tickets/screenshots (Section 10) — listed only as a Phase 24 test item, but containment is an architecture property (data/instruction segregation in context construction) that must be designed in Phase 13, not tested-in at 24.
The corpus itself is crown-jewel IP — full PREMIA source, dependencies, and error semantics per customer in one SQLite file. Encryption at rest, file-level access control, and export controls on snapshot files need explicit treatment; a snapshot file walking out the door is a total compromise of that customer's codebase.
Evidence/audit integrity — the ledger and audit trail are only trustworthy if append-only is enforced below the application layer (DB permissions, hash chaining) — otherwise "immutable" is a comment in the code.
LLM egress in on-prem cells — is the LLM provider local or external? If external, ticket text, source code, and possibly probe results transit out of the customer's network. This decision is absent from the plan and several customers will force it. Make provider locality a per-cell policy decision now.
15. Evaluation Assessment — Decision L: ADOPT, MODIFY

W — Sufficient methodology? The metric list is good; the methodology around it is incomplete: no frozen eval snapshots (eval must pin snapshot and toolchain versions), no contamination control (Section 6), no difficulty stratification, and no negative controls — golden records where the correct output is "insufficient evidence" or "not reproducible from static analysis." Without negative controls you cannot measure over-claiming, which is this product's cardinal sin.

X — Gameable metrics: Recall@K inflates with K — fix K per metric and report the curve. Unsupported-claim rate is gamed by making fewer claims — pair it with a claim-coverage/completeness measure against the golden failure path. Citation validity is gamed by citing everything — measure citation precision, not just validity. SME validation time is confounded by SME familiarity and ticket difficulty — treat as observational, never as an optimization target. RCA correctness needs a rubric (root cause identified / contributing factors / resolution correctness scored separately), or SME grading will be incomparable across SMEs.

16. Phase Plan Assessment

Individually reasonable, collectively mis-ordered. The three sequencing errors: ledger after agent (Section 1), SME-validation/evaluation after everything (Section 1), storage benchmark (22) after the schemas that depend on the store (10, 16). Additional: 16 sequential phases is a waterfall in phase-numbered clothing — the plan needs one thin vertical slice early (real ticket → retrieval → RCA → SME validates → golden record persisted) so every subsequent phase is measured against accumulating golden data. Also move PLL/PLD ingestion forward out of Phase 12 — negative-evidence soundness depends on it (Section 2), and negative evidence is load-bearing for the verifier.

17. Critical Risks (top 6, unresolved)
Identity migration (H4/T): re-keying 1,091 artifacts and 244k dependencies without breaking existing provenance and tests. Highest technical risk in Phase 9.
Verifier over-trust: organization and customers reading "citation-verified" as "correct." A product/UX risk with architectural roots.
Corpus incompleteness poisoning negative evidence until PLL/RDF coverage lands.
Evaluation contamination via historical-RCA retrieval, silently inflating every metric.
Probe-result data governance absent for an insurance-domain product.
Agent value unproven — risk of building Phase 15 because it's exciting, when deterministic retrieval expansion may close the gap at a tenth of the cost.
18. Decisions That Need Experiments
SQLite vs PostgreSQL for the operational store only (concurrent writers + SKIP LOCKED queue under worker load).
Bounded agent vs deterministic retrieval-expansion fallback, on gate-failure tickets, measured on golden data.
Vector retrieval — only after lexical/structural recall is measured and found insufficient.
OCR pipeline choice — extraction P/R on a real attachment sample.
Full-reindex cost under analyzer version bump (post-H2 behavior) — operational feasibility measurement.
19. Decisions That Should Be Locked
Deterministic core as sole source of truth; LLM never originates facts.
Existing RCA engine preserved (A).
Content-addressed canonical artifact identity: identity = (canonical repository-relative path, content hash), with derived-artifact identity = (source identity, toolchain version) — never scan order, never insertion order (answers S/T: current identities are not suitable for immutable evidence; this is).
Snapshot = source content set + toolchain version vector; investigations pin snapshots (answers U/V: pinning is sufficient only under this definition; additional provenance required = toolchain versions, retrieval configuration, model/prompt versions per LLM call).
CKG as tool/query contract over the existing index; no second graph store (C as modified).
Corpus store: SQLite file-per-snapshot. Operational store: separate, PG-pending-benchmark.
Ledger: append-only, supersedes-links, class-transition rules; findings cite evidence IDs.
Verifier scope = citation integrity; downgrade-not-reject; never labeled as correctness.
Runtime: templates-only (no free SQL path exists at all), parser-validated at authoring, isolated executor process, read-only per-env credentials, human approval per probe, permanent no-autonomous-execution.
Ticket content is untrusted data in all LLM contexts; write-back human-approved per action.
Cells as the isolation model; customer/environment IDs in schema from Phase 10.
Eval snapshots frozen; golden data partitioned from retrieval; negative controls included.
PG-backed job queue; no Kafka/RabbitMQ/Neo4j/microservices at cell scale.
20. Decisions That Should Be Deferred

Bounded agent (until experiment, Section 7). Vector retrieval. Multi-customer control plane. Full explorer suite (build the SME-validation slice first; explorers follow). RDF/Reports/documentation parsers (PLL/PLD moves up, the rest stays deferred). Fine-tuning (correctly never proposed). Kafka, Kubernetes, Neo4j, microservices extraction (interfaces only).

21. Proposed Changes to the Plan
Phase 9 gains an identity specification deliverable (canonical ID scheme + migration/aliasing plan) as a precondition of the H4 fix.
Insert a thin vertical slice with minimal SME validation UI immediately after the domain model, before agent/retrieval expansion.
Move Evidence Ledger + verifier before any agent work; move the operational-store benchmark before ledger persistence.
Pull PLL/PLD ingestion forward; add corpus-scope qualifiers to negative evidence in the interim.
Reframe verifier outputs as citation integrity throughout schema, API, and UI.
Add probe-result data governance and LLM-locality policy to the runtime and cell designs.
Demote Phase 15 (agent) to experiment-gated.
22. Revised Final Architecture
                        SUPPORT UI (Ticket · Investigation · RCA · SME Validation · Explorers)
                                              │
                                      APPLICATION API
                                              │
        ┌───────────────┬─────────────────────┼──────────────────────┬───────────────┐
   Ticket Intelligence  │            Investigation Engine            │        Evaluation
   (Jira mirror, OCR,   │   sufficiency gate → bounded RCA call      │   (golden records,
   signals; untrusted-  │   └─ gate-fail → [experiment: agent /      │    frozen snapshots,
   data containment)    │        deterministic expansion]            │    replay, metrics)
                        │                     │                      │
                        │          Evidence Ledger (append-only)     │
                        │            + Citation Verifier             │
                        │                     │                      │
                 CKG TOOL CONTRACT  (search_symbols … find_error_sites)
                        │                                            │
        ┌───────────────┴────────────┐                 ┌─────────────┴────────────┐
        CORPUS STORE                                    OPERATIONAL STORE
        SQLite, one file per snapshot                   PostgreSQL (pending narrow
        (artifacts, symbols, deps, CFG,                 benchmark): customers, envs,
        FTS, errors) — immutable,                       tickets, investigations,
        content-addressed identity                      evidence, jobs, audit, users
                        │
        Ingestion workers (FMB→XML as isolated component, parsers, indexing)
                        
        Runtime Executor — separate process, templates-only, read-only creds,
        human-approved, audited, result-data governance    →  RUNTIME_CONFIRMED evidence
                        
        Deployment unit: one CELL per customer (on-prem), environments within a cell

One sentence of explanation: everything the AI sees flows through the CKG tool contract over an immutable, content-addressed corpus snapshot; everything the AI says flows through the ledger and citation verifier into a human-validated golden loop; nothing autonomous touches customer systems.

23. Revised Phase Sequence

9 — Integrity + identity spec (below) → 10 — Domain model (with customer/env IDs; operational-store benchmark decided here) → 10.5 — Thin vertical slice + minimal SME validation workflow; golden records begin accumulating → 11 — CKG tool contract (adapter, no new store) → 12a — PLL/PLD coverage + negative-evidence scoping → 13 — Ticket intelligence (with injection containment) → 16 — Ledger + citation verifier (moved up) → 14 — Retrieval 2.0 (measured against golden data) → 15 — Agent experiment, then build only on positive result → 17 — Runtime integration (with data governance) → 18 — Jira write-back → 19 — Full UI/explorers → 20 — Knowledge overlays → 12b — Remaining parsers → 21 — Cell/control-plane maturation → 23 — Evaluation platform hardening → 24 — Production hardening (continuous items pulled forward where noted).

24. Go/No-Go Conditions for Phase 9 — and PHASE 9 EXACT SCOPE

GO, conditional on the amended scope below.

Phase 9 MUST implement:

Canonical identity specification: (path, content-hash) artifact IDs; (source ID, toolchain version) derived IDs; migration/alias table preserving old-ID resolution; provenance continuity test on the real corpus.
H1: resolve retrieval candidates to graph-bearing source entities before CFG availability checks; regression test from the already-reproduced real-corpus case.
H2: toolchain version vector in the derivation key; version bump triggers regeneration; measure full-regeneration cost (feeds Section 18).
H3: SKIPPED/OK contract between pipeline and indexer defined and tested with a second-run-unchanged regression test.
H5: structured error taxonomy for provider/CLI failures; no raw exception escapes the boundary.
H6: remove the free-SQL path entirely — templates-only enforcement + parser-based template validation at authoring; fail-closed regression tests including the quoted-identifier bypass.
High-impact mediums from the review, plus: freeze the regression corpus and record its content hash as the Phase 9 baseline.

Phase 9 MUST NOT touch: any new domain objects (Investigation, Ledger, Ticket), any schema for the operational store, any PostgreSQL work, the agent, CKG adapter, Jira, UI, retrieval expansion, or vector anything. Phase 9 is purely: make the existing deterministic core internally consistent, identity-stable, and fail-closed, with the full regression suite green against the frozen corpus.

Implementation readiness: the converged architecture is ready for Phase 9 as amended — not as originally written. The core bet (preserve the deterministic engine, wrap it in Tracer's product concepts, migrate storage and add agency only on evidence) is correct. The plan fails as written on sequencing (ledger/eval/identity ordering) and on two honesty gaps (verifier scope, negative-evidence scope), all of which are correctable on paper before any Phase 9 code is written — which is exactly when to correct them.

If you'd like, I can put this report into a Markdown or Word file for circulation.