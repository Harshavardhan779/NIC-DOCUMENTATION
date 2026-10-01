# NIC AI PREMIA Support Intelligence Platform
## Master Product, Capability, Architecture & AI Operating Model

> **Purpose:** This document is the common reference for humans and AI agents. It explains what we are building, why it exists, the agreed product capabilities, deterministic intelligence, AI/LLM role, agent model, expected outputs, SQL Chatbot, Application Intelligence, security, and target architecture.
>
> **Implementation rule:** This document describes the target product. It does **not** override the currently approved implementation specifications, plans, tasks, tests, reviews, or acceptance gates. New capabilities must enter the approved implementation flow before code changes are made.

---

# 1. Executive Summary

We are building an **AI-assisted PREMIA Support Intelligence Platform**.

The product helps PREMIA support engineers investigate customer incidents by connecting:

- Jira incidents
- comments/history
- screenshots and attachments
- Oracle Forms
- Forms XML
- PL/SQL/application logic
- dependencies
- control-flow analysis
- database intelligence
- PREMIA domain knowledge
- deterministic evidence
- AI reasoning
- optional runtime evidence
- SME validation

This is **not a generic chatbot**.

It is an **evidence-driven support investigation platform**.

The core trust model is:

```text
Deterministic analysis = SOURCE OF TRUTH
LLM / AI              = REASONING + HYPOTHESIS
Runtime evidence      = CONFIRMATION
SME                   = FINAL VALIDATION
```

---

# 2. Problem We Are Solving

A PREMIA support investigation can require a support engineer to move between Jira, screenshots, Oracle Forms, XML, PL/SQL, dependencies, database objects, configuration, and historical knowledge.

The current manual pattern is:

```text
Jira ticket
   ↓
Understand issue
   ↓
Find screen
   ↓
Find Form
   ↓
Search source
   ↓
Trace dependencies
   ↓
Trace database logic
   ↓
Understand error
   ↓
Prepare RCA
```

The target pattern is:

```text
Jira incident
   ↓
Ticket understanding
   ↓
PREMIA area identification
   ↓
Application Intelligence
   ↓
Deterministic evidence
   ↓
AI-assisted reasoning
   ↓
Evidence-backed RCA
   ↓
Optional runtime validation
   ↓
SME decision
```

---

# 3. Product Identity

**NIC AI PREMIA Support Intelligence Platform**

The final product combines:

```text
Enterprise Support Console
+
Oracle Forms Intelligence
+
Application Intelligence
+
Database Intelligence
+
Dependency / Control-Flow Analysis
+
PREMIA Knowledge
+
Evidence-Based RCA
+
AI Reasoning
+
Optional Runtime Verification
+
SME Review
```

The existing deterministic RCA engine is the foundation. The application should build around it rather than replace it with an LLM.

---

# 4. What the Product Is NOT

The platform is not:

- a generic ChatGPT-style chatbot
- an unrestricted autonomous agent
- an LLM that independently decides what source to retrieve
- an LLM that invents dependencies
- an LLM that invents database state
- an autonomous production modification system
- an uncontrolled agent loop
- a system where every AI answer is treated as fact

The product must remain deterministic wherever determinism is possible.

---

# 5. Evidence Classification

Every important result should be classified as:

```text
OBSERVED
DERIVED
HYPOTHESIS
RUNTIME_CONFIRMED
UNRESOLVED
```

**Observed:** directly present in source/ticket/deterministic evidence.

**Derived:** logically derived from authoritative evidence.

**Hypothesis:** AI-generated explanation requiring validation.

**Runtime confirmed:** established by an executed runtime diagnostic.

**Unresolved:** insufficient evidence.

A hypothesis must never be presented as an observed fact.

---

# 6. End-to-End Product Flow

```text
Customer / Environment
        ↓
JIRA INCIDENT
        ↓
Ticket Intelligence
        ↓
PREMIA Area Identification
        ↓
Application Intelligence
        │
        ├── Forms / XML
        ├── Blocks / Items
        ├── Triggers
        ├── Program Units
        ├── Dependencies
        ├── CFG / Control Flow
        ├── Errors
        └── Database References
        ↓
Database Intelligence
        ↓
Deterministic Evidence
        ↓
RCA Context / Sufficiency Gate
        ↓
AI Reasoning
        ↓
RCA Hypothesis
        ↓
Deterministic Validation
        ↓
Optional Read-only Runtime Evidence
        ↓
SME Review
        ↓
Support Decision
```

---

# 7. Major Product Capabilities

## 7.1 Customer & Environment Management

The platform is intended to support multiple customers and multiple environments.

```text
Customer
   ↓
Environment
   ↓
Configuration
   ↓
Artifacts / Database / Jira
   ↓
Indexes
   ↓
Evidence
   ↓
RCA
```

Every investigation is scoped by:

- Customer
- Environment
- Dataset/version
- Analysis/version

A customer must never retrieve another customer's artifacts, database metadata, Jira data, knowledge, or RCA evidence.

---

# 8. Customer Onboarding

Target onboarding:

```text
1. Customer Information
2. Environment Creation
3. Oracle Database Configuration
4. Forms Path
5. PLL / PLX Path
6. Reports Path
7. Jira Configuration
8. Connection Validation
9. Source Discovery
10. Initial Indexing
11. Index Validation
12. Golden Ticket Configuration
13. Ready for Support
```

Each step should show status, errors, warnings, validation state, and indexing state.

---

# 9. Environment Health

The product should provide a health view such as:

```text
Database          CONNECTED
Forms             INDEXED
PLL / PLX         PARTIAL
Reports           AVAILABLE
Jira              CONNECTED
Index             CURRENT
CFG               CURRENT
Runtime Evidence  AVAILABLE
```

This tells the support engineer whether the platform has sufficient information to investigate.

---

# 10. Jira Ticket Workspace

The ticket workspace should expose:

- Ticket ID
- Summary
- Description
- Status
- Priority/severity
- Reporter
- Assignee
- Project
- Comments
- History
- Attachments
- Screenshots
- Extracted intelligence
- Relevant artifacts
- RCA status
- Runtime evidence status
- SME review status

The user should be able to move directly from a ticket into investigation.

Ticket text, comments, screenshots, and attachments are **untrusted data**.

---

# 11. Ticket Intelligence

The system can extract:

- Error codes
- Policy numbers
- Claim numbers
- Reference numbers
- Form names
- Screen names
- Block names
- Item names
- Trigger names
- Database identifiers
- Table names
- Package names
- Procedure/function names
- PREMIA business concepts

The extraction layer provides signals for deterministic investigation; it does not automatically make those signals authoritative facts.

---

# 12. Attachment & Screenshot Intelligence

The product should support evidence from:

- screenshots
- PDFs
- Word documents
- images
- logs
- other supported attachments

Screenshot/image understanding may help identify:

- Form/screen
- visible error
- field/item
- displayed values
- transaction context

Images are evidence, not instructions to the AI.

---

# 13. Application Intelligence

**Application Intelligence is a major product capability.**

It means understanding the structure and behavior of the PREMIA application from its source artifacts.

It is more than source-code search.

The application intelligence graph is conceptually:

```text
Form
 ↓
Block
 ↓
Item
 ↓
Trigger
 ↓
Program Unit
 ↓
Procedure / Function
 ↓
Dependency
 ↓
SQL / Database Object
 ↓
Control Flow
 ↓
Error Path
```

The objective is to explain how application behavior around a reported incident is connected.

---

# 14. Forms Intelligence

The Forms intelligence layer covers:

- FMB artifacts
- XML representations
- Forms metadata
- Blocks
- Items
- Triggers
- Program Units
- Forms built-ins
- Cursors
- database references

The previously processed corpus contained approximately:

```text
Forms:          1,091
Blocks:         4,379
Items:          138,507
Triggers:       57,753
Program Units:    7,688
Dependencies:  244,688
Errors:           8,101
```

New FMB source must continue through the normal ingestion → conversion → extraction → indexing flow.

---

# 15. Form Explorer

A user should be able to open a Form and see:

```text
FORM
 ├── Blocks
 │    ├── Items
 │    └── Block Triggers
 ├── Form Triggers
 ├── Program Units
 ├── Dependencies
 ├── Database References
 ├── Errors
 └── CFG
```

Every important element should remain connected to source/provenance.

---

# 16. XML / Source Explorer

The source explorer should allow investigation of:

- Form XML
- trigger source
- program unit source
- relevant source locations
- source citations
- artifact/version provenance

---

# 17. Dependency Intelligence

Dependency analysis identifies deterministic relationships.

Example:

```text
WHEN-VALIDATE-ITEM
        ↓
CHECK_POLICY_STATUS
        ↓
PKG_POLICY_VALIDATION
        ↓
POLICY_TABLE
```

The platform should allow navigation through these relationships.

The AI must not create a dependency that the deterministic layer did not establish.

---

# 18. Control-Flow Intelligence

CFG provides deterministic execution-path evidence.

It supports concepts including:

- IF / ELSIF / ELSE
- BEGIN / EXCEPTION
- LOOP / EXIT
- cursor operations
- %FOUND / %NOTFOUND
- RAISE
- RAISE_APPLICATION_ERROR
- procedure/function calls
- SQL
- error paths

The goal is to answer questions such as:

> What deterministic execution path can lead from this trigger to this error?

---

# 19. Database Intelligence

Database Intelligence connects application logic to database structures:

```text
Application
   ↓
Procedure / Function
   ↓
Package
   ↓
SQL
   ↓
Table / View
   ↓
Relevant columns / relationships
```

Capabilities may include:

- table exploration
- column exploration
- package/procedure exploration
- SQL relationships
- application-to-database references
- database dependency analysis
- metadata inspection
- read-only runtime verification

All database intelligence is customer/environment scoped.

---

# 20. SQL Chatbot

The SQL chatbot is an **interactive database intelligence interface**, not an unrestricted production SQL executor.

Target flow:

```text
Natural-language question
        ↓
SQL intelligence
        ↓
SQL proposal
        ↓
Safety / validation
        ↓
Read-only execution
        ↓
Database result
        ↓
Explanation
```

Example questions:

```text
"Show the policy details for this policy number."

"Does this claim exist?"

"Which records are associated with this claim?"

"Show the current status of this policy."

"Which database objects are involved in this application flow?"
```

The SQL chatbot must not:

- modify production data
- execute destructive SQL
- bypass read-only controls
- expose credentials
- fabricate database results
- treat generated SQL as automatically trusted

The intended runtime database account is read-only.

---

# 21. PREMIA Knowledge

PREMIA Knowledge combines:

```text
GLOBAL / COMMON PREMIA KNOWLEDGE
+
CUSTOMER-SPECIFIC KNOWLEDGE
+
ENVIRONMENT-SPECIFIC INFORMATION
```

### Common

- PREMIA concepts
- standard terminology
- standard Forms relationships
- standard business concepts

### Customer-specific

- custom Forms
- custom packages
- custom workflows
- custom rules
- customer terminology

### Environment-specific

- installed artifacts
- database objects
- configuration
- runtime information

The origin of knowledge must remain distinguishable.

---

# 22. RCA Workspace

The RCA workspace combines all evidence:

```text
Ticket Evidence
Screenshot Evidence
Form Evidence
Dependency Evidence
CFG Evidence
Database Evidence
PREMIA Knowledge
        ↓
Deterministic Findings
        ↓
AI Reasoning
        ↓
RCA Hypothesis
        ↓
Validation
        ↓
SME Decision
```

The workspace must show **why** a conclusion was produced.

---

# 23. Expected RCA Output

Conceptual output:

```text
Relevant Form:
    <deterministic result>

Relevant Item:
    <deterministic result>

Relevant Trigger:
    <deterministic result>

Control-Flow Path:
    <deterministic CFG evidence>

Relevant Procedures / Functions:
    <deterministic dependency evidence>

Relevant Database Objects:
    <deterministic references>

Relevant Errors:
    <deterministic error evidence>

Static Evidence:
    <citations>

AI RCA Hypothesis:
    <evidence-constrained reasoning>

Runtime Status:
    NOT_PERFORMED / CONFIRMED / CONTRADICTED

SME Validation:
    REQUIRED
```

The system must never claim production behavior without runtime confirmation.

---

# 24. AI / LLM Role

The LLM receives bounded evidence:

```text
Ticket Intelligence
+
Deterministic Evidence
+
Relevant PREMIA Knowledge
+
Optional Runtime Evidence
```

It produces:

```text
Reasoning
+
Hypothesis
+
Explanation
+
Missing Evidence
+
Recommended Validation
```

It does not determine authoritative source retrieval.

---

# 25. Expected AI Model Strategy

The platform should support multiple model options:

```text
ON-PREMISE / LOCAL LLM
        +
ENTERPRISE LLM PROVIDER
        +
CUSTOMER-SPECIFIC MODEL
```

The platform should use an LLM abstraction rather than hardcoding one model/provider.

Model evaluation should consider:

- reasoning quality
- evidence adherence
- unsupported-claim rate
- hallucination rate
- latency
- context capacity
- security/deployment constraints
- cost
- reproducibility

The production model should be selected through evaluation, not assumption.

---

# 26. External AI Feasibility Test

Before making an external AI model part of the product, we can evaluate the deterministic output independently.

```text
Ticket
   ↓
Existing Deterministic Layer
   ↓
Deterministic Output
   ↓
External LLM
   ↓
RCA Hypothesis
```

No project API call is required.

The key question is:

> **Does our deterministic layer provide enough evidence for an LLM to produce useful, evidence-constrained RCA?**

This is an evaluation activity, not a replacement for the deterministic architecture.

---

# 27. Agent Model

Agents/components may be used for bounded responsibilities.

Conceptually:

```text
                 SUPPORT INVESTIGATION
                         ↓
                  ORCHESTRATION
                         ↓
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
Ticket Intelligence  Application Intel  Database Intel
       └─────────────────┼─────────────────┘
                         ↓
                Deterministic Evidence
                         ↓
                    RCA Reasoning
                         ↓
                     Validation
```

Potential bounded roles:

### Ticket Intelligence Agent
Structures ticket signals.

### Application Investigation Agent
Navigates authorized deterministic application evidence.

### Database Investigation Agent
Works with approved database metadata/read-only diagnostics.

### RCA Reasoning Agent
Synthesizes evidence into hypotheses.

### Validation Agent
Checks generated claims against authoritative evidence.

These are **bounded components, not unrestricted autonomous decision makers**.

---

# 28. Runtime Evidence

Runtime evidence can confirm facts that static analysis cannot establish.

Examples:

- policy record verification
- claim record verification
- configuration lookup
- procedure/package metadata
- relevant database state

Rules:

```text
READ ONLY
EXPLICIT
DETERMINISTIC
AUDITABLE
NO PRODUCTION MUTATION
```

Static and runtime evidence must remain distinguishable.

---

# 29. SME Review

Target workflow:

```text
AI Hypothesis
    ↓
Evidence Review
    ↓
Optional Runtime Validation
    ↓
SME Review
    ↓
Accepted / Rejected / Needs Investigation
```

Only validated SME outcomes should become trusted evaluation/knowledge data.

---

# 30. Golden Dataset & Evaluation

A historical-ticket evaluation dataset should contain:

- Jira ticket
- comments
- screenshots/attachments
- expected Form
- expected Item
- expected Trigger
- expected database objects
- known RCA
- known resolution
- SME validation

Metrics can include:

- Form retrieval accuracy
- Item retrieval accuracy
- Trigger retrieval accuracy
- Database-object retrieval accuracy
- failure-path accuracy
- RCA accuracy
- unsupported-claim rate
- runtime-confirmation accuracy
- end-to-end latency

This measures whether the system is becoming a real support agent rather than only a code-search system.

---

# 31. Search Experience

Support exact and natural-language search across:

- Jira tickets
- Forms
- Blocks
- Items
- Triggers
- Program Units
- Errors
- Tables
- Packages
- Procedures
- Functions

Results should explain why each result was retrieved.

---

# 32. Evidence & Provenance

For every RCA investigation, the platform should be able to answer:

```text
Who initiated it?
When?
Which customer?
Which environment?
Which ticket?
Which dataset?
Which source artifacts?
Which artifact version?
Which RCA engine version?
Which CFG version?
Which prompt version?
Which LLM/model?
What evidence was supplied?
What runtime evidence was executed?
What did validation accept/reject?
What did the SME decide?
```

This provides reproducibility and auditability.

---

# 33. Failure States

The product must explicitly handle:

- Jira unavailable
- Jira authentication failure
- attachment unavailable
- image processing failure
- no relevant Form
- multiple candidate Forms
- insufficient evidence
- ambiguous evidence
- CFG unavailable
- database unavailable
- LLM unavailable
- LLM validation failure
- runtime not executed
- runtime execution error
- stale index
- failed artifact conversion
- missing source
- missing dependency
- SME review pending

The UI must explain why the system cannot proceed.

---

# 34. Roles

Potential roles:

- Platform Administrator
- Customer Administrator
- Support Engineer
- Developer
- DBA
- SME
- Viewer
- Evaluation/QA User

Permissions should control:

- customer configuration
- environment configuration
- Jira configuration
- source indexing
- database access
- RCA execution
- runtime evidence
- SME validation
- knowledge management
- evaluation

---

# 35. Security

Core requirements:

- tenant isolation
- environment isolation
- least privilege
- secure secrets management
- read-only runtime database access
- no credentials in prompts
- no credentials in logs
- no credentials in RCA evidence
- no credentials in screenshots
- no credentials in audit output
- controlled LLM egress
- untrusted ticket-content boundary
- auditability

Enterprise deployments may require customer-controlled/on-premise operation.

---

# 36. Deployment Model

The architecture should support:

- on-premise deployment
- customer-controlled infrastructure
- customer-controlled Oracle connectivity
- customer-controlled Jira connectivity
- source code remaining inside customer environment
- no production database writes
- local/on-premise LLM deployment where required
- customer-controlled secrets
- audit logging
- RBAC

The architecture must not assume customer data can freely leave the customer environment.

---

# 37. Multi-Customer Architecture

Target model:

```text
GENERIC PLATFORM
+
CUSTOMER CONFIGURATION
+
CUSTOMER SOURCE DATA
+
CUSTOMER JIRA
+
CUSTOMER DATABASE
+
CUSTOMER KNOWLEDGE
```

NIC's PREMIA environment is the first configured customer/environment.

The architecture should support additional customers without rewriting the core system.

---

# 38. Incremental Processing

If one Form changes:

```text
Changed Artifact
      ↓
Detect Change
      ↓
Reprocess Artifact
      ↓
Update Dependencies
      ↓
Update Affected CFG
      ↓
Update Relevant Index
```

The same principle applies to:

- Forms
- PLL/PLX
- Reports
- database metadata
- knowledge
- Jira information

Avoid unnecessary full-corpus reprocessing.

---

# 39. Versioning

Version:

- source dataset
- artifact versions
- index versions
- CFG analyzer
- RCA engine
- prompt
- LLM model/version
- PREMIA knowledge
- runtime evidence rules
- evaluation dataset

An RCA should be reproducible against the versions that generated it.

---

# 40. Observability

Track:

- request latency
- Jira API latency
- attachment processing time
- retrieval latency
- CFG generation time
- LLM latency
- runtime evidence latency
- end-to-end RCA latency
- indexing duration
- job duration
- failure rates

Use correlation/request IDs.

---

# 41. Modular Architecture

Conceptual backend modules:

```text
Tenant / Customer
Environment
Configuration
Jira
Ticket Intelligence
Attachments
Image Evidence
Artifacts
Forms
PLL / PLX
Reports
Database
Dependencies
CFG
Deterministic Retrieval
RCA Context
LLM Reasoning
RCA Validation
Runtime Evidence
PREMIA Knowledge
SME Review
Evaluation
Audit
Security / Secrets
Application API
```

Each module should have:

- clear responsibility
- public interface
- explicit inputs
- explicit outputs
- minimal dependencies
- test boundary

Avoid god modules.

---

# 42. Frontend Architecture

Suggested feature boundaries:

```text
/tickets
/forms
/dependencies
/database
/rca
/runtime
/knowledge
/evaluation
/admin
```

Shared:

```text
/components
/hooks
/services
/types
/utils
```

---

# 43. Backend Architecture

Conceptual separation:

```text
/domain
/application
/infrastructure
/interfaces
```

Use a **modular monolith first**.

Extract services only when actual scaling or operational requirements justify it.

Do not introduce distributed infrastructure simply for architectural appearance.

---

# 44. Scalability

Target scale:

```text
Customers
    1 → 10 → 100+

Environments
    multiple per customer

Artifacts
    hundreds → thousands → tens of thousands

Tickets
    hundreds → millions

RCA requests
    one user → many concurrent support engineers
```

Use, where justified:

- incremental processing
- asynchronous jobs
- bounded LLM contexts
- efficient indexing
- customer/environment partitioning
- observable workloads
- later service extraction

---

# 45. Performance UX

Long-running operations should expose meaningful states:

```text
Ticket Retrieval       RUNNING
Attachment Processing  72%
Artifact Indexing      COMPLETE
CFG Generation         54%
RCA Generation         WAITING
Runtime Evidence       NOT EXECUTED
```

Supported states:

```text
RUNNING
WAITING
COMPLETED
FAILED
PARTIAL
BLOCKED
```

---

# 46. Application Intelligence + SQL Chatbot

These capabilities should work together:

```text
Jira Incident
      ↓
Application Intelligence
      ↓
Relevant Form
      ↓
Trigger
      ↓
Procedure / Function
      ↓
Database Object
      ↓
Database Intelligence
      ↓
SQL Chatbot / Read-only Query
      ↓
Runtime Confirmation
      ↓
RCA
```

This creates an application-to-database investigation path.

---

# 47. Expected User Experience

A support engineer starts with a statement such as:

> "Policy approval is taking FAC special retention even though it is within treaty limit."

The platform progressively exposes:

```text
Incident Understanding
        ↓
Relevant PREMIA Area
        ↓
Relevant Form / Screen
        ↓
Relevant Item / Trigger
        ↓
Application Logic
        ↓
Dependencies
        ↓
Database Flow
        ↓
Deterministic Evidence
        ↓
AI Hypothesis
        ↓
Validation
        ↓
SME Decision
```

The engineer should not have to reconstruct the entire application flow manually across disconnected tools.

---

# 48. Final Product Output

Conceptually:

```text
┌────────────────────────────────────────────────────┐
│                 SUPPORT INCIDENT                   │
├────────────────────────────────────────────────────┤
│ Ticket Understanding                              │
│ Relevant PREMIA Area                              │
│ Relevant Form / Screen                            │
│ Relevant Item / Trigger                           │
│                                                    │
│ APPLICATION INTELLIGENCE                          │
│  ├─ Source                                        │
│  ├─ Dependencies                                  │
│  ├─ CFG                                           │
│  └─ Errors                                        │
│                                                    │
│ DATABASE INTELLIGENCE                             │
│  ├─ Tables                                        │
│  ├─ Packages                                      │
│  └─ Read-only Runtime Evidence                    │
│                                                    │
│ RCA                                                │
│  ├─ Deterministic Evidence                        │
│  ├─ AI Hypothesis                                 │
│  ├─ Missing Evidence                              │
│  └─ Recommended Validation                        │
│                                                    │
│ SME REVIEW                                         │
└────────────────────────────────────────────────────┘
```

---

# 49. Current Deterministic Foundation

The existing foundation includes:

- project ingestion
- artifact registry
- FMB → XML conversion
- Forms extraction
- SQLite indexing
- deterministic retrieval
- dependency analysis
- CFG/control-flow analysis
- RCA evidence construction
- retrieval sufficiency gate
- bounded RCA context
- LLM reasoning interface
- deterministic validation
- runtime evidence foundation

The product architecture builds around this foundation.

---

# 50. Product Evolution

The system is evolving from:

```text
NIC AI RCA POC
```

toward:

```text
NIC AI PREMIA SUPPORT AGENT
```

The RCA engine remains a core backend capability.

The final product is broader:

```text
SUPPORT APPLICATION
+
APPLICATION INTELLIGENCE
+
DATABASE INTELLIGENCE
+
PREMIA KNOWLEDGE
+
RCA
+
AI REASONING
+
RUNTIME VALIDATION
+
SME REVIEW
```

---

# 51. Product Differentiator

The differentiator is not simply adding an LLM.

The differentiator is:

> **AI reasoning grounded in deterministic understanding of the customer's PREMIA application.**

The platform first establishes:

```text
WHAT EXISTS
WHAT IS CONNECTED
WHAT PATHS EXIST
WHAT DATABASE OBJECTS ARE REFERENCED
WHAT ERROR PATHS EXIST
WHAT EVIDENCE SUPPORTS THE FINDING
```

Then AI reasons over that evidence.

---

# 52. Target Architecture

```text
                    NIC AI PREMIA SUPPORT AGENT
                               │
          ┌────────────────────┴────────────────────┐
          │                                         │
   SUPPORT APPLICATION                       DETERMINISTIC CORE
          │                                         │
   ┌──────┼────────┐                      ┌─────────┼─────────┐
   │      │        │                      │         │         │
 Jira   Forms   Database               Ingestion  Index   Retrieval
   │      │        │                      │         │         │
Ticket Explorer SQL Chatbot           FMB→XML    CFG    Evidence
Intel                                      Parsing        Context
   │      │        │                      │         │         │
   └──────┼────────┘                      └─────────┼─────────┘
          │                                         │
          └────────────────────┬────────────────────┘
                               ▼
                       PREMIA KNOWLEDGE
                               │
                               ▼
                         AI REASONING
                               │
                               ▼
                      RCA VALIDATION
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
           Runtime Evidence            SME Review
                  │                         │
                  └────────────┬────────────┘
                               ▼
                       SUPPORT DECISION
```

---

# 53. Non-Negotiable Principles

1. Deterministic analysis is authoritative.
2. LLM reasoning is bounded by supplied evidence.
3. Runtime evidence is separate from static evidence.
4. SME remains final validation authority.
5. No fabricated source or dependency relationships.
6. No fabricated runtime facts.
7. No autonomous production mutation.
8. Runtime database access is read-only.
9. Customer/environment isolation is mandatory.
10. Ticket and attachment content is untrusted input.
11. Important findings must have provenance.
12. AI hypotheses must be distinguishable from facts.
13. Indexes and analysis must be versioned.
14. Historical tickets become trusted evaluation data only after validation.
15. Modular monolith first; service extraction only when justified.
16. Do not over-engineer.
17. Use deterministic methods wherever possible.
18. Implementation must follow the approved task/spec/plan flow.

---

# 54. One-Sentence Definition

> **NIC AI PREMIA Support Intelligence Platform is an evidence-driven support system that connects Jira incidents with deterministic understanding of PREMIA Forms, application logic, dependencies and databases, then uses bounded AI reasoning, optional runtime verification and SME validation to produce explainable RCA and support decisions.**

---

# 55. One-Slide Executive Version

## WHAT ARE WE BUILDING?

An **AI-assisted PREMIA Support Intelligence Platform** for investigating customer incidents.

## WHAT DOES IT CONNECT?

```text
Jira
+
Screenshots / Attachments
+
PREMIA Forms
+
Application Logic
+
Dependencies
+
Database
+
PREMIA Knowledge
+
AI
+
Runtime Evidence
```

## WHAT DOES IT PRODUCE?

```text
Incident Understanding
        ↓
PREMIA Area
        ↓
Application / Database Evidence
        ↓
Evidence-backed RCA Hypothesis
        ↓
Validation
        ↓
SME Decision
```

## WHAT MAKES IT DIFFERENT?

**Deterministic evidence first. AI reasoning second.**

## FINAL VISION

Transform PREMIA support from manual investigation across multiple systems into **one evidence-driven AI-assisted support investigation workspace**.

---

# 56. Implementation Discipline

This document is the common product reference.

It does **not** authorize skipping the current implementation process.

Implementation remains:

```text
Approved Architecture
        ↓
Approved Specification
        ↓
Approved Implementation Plan
        ↓
Task
        ↓
TDD
        ↓
Targeted Tests
        ↓
Full Regression
        ↓
Security / Boundary Verification
        ↓
Milestone Review
        ↓
Acceptance
        ↓
Next Task
```

New capabilities such as Application Intelligence, SQL Chatbot, additional agent capabilities, new models, or UI capabilities must enter the appropriate approved specification/plan before implementation.

---

# 57. Final Mental Model

```text
                       SUPPORT INCIDENT
                              │
                              ▼
                         JIRA / TICKET
                              │
                              ▼
                    TICKET INTELLIGENCE
                              │
                              ▼
                   PREMIA INTELLIGENCE
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
       Forms/XML        Application Logic      Database
          │                   │                   │
          ▼                   ▼                   ▼
      Triggers            Dependencies        SQL/Objects
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                    DETERMINISTIC EVIDENCE
                              │
                              ▼
                       RCA CONTEXT / GATE
                              │
                              ▼
                          AI / LLM
                    Reasoning + Hypothesis
                              │
                              ▼
                    DETERMINISTIC VALIDATOR
                              │
                              ▼
                     RUNTIME VERIFICATION
                         (READ ONLY)
                              │
                              ▼
                         SME REVIEW
                              │
                              ▼
                      SUPPORT DECISION
```

**This is the product we are building.**
