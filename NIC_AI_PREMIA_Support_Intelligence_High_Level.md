# NIC AI — PREMIA Support Intelligence Platform
## High-Level Product Story

---

## 1. What Are We Building?

### An AI-Assisted PREMIA Support Intelligence Platform

We are building a system that helps support engineers **investigate PREMIA incidents faster and with evidence**, instead of manually searching through Forms, source code, database logic, and support information.

### Core Idea

> **Give the system a support ticket → understand the incident → identify where the issue is occurring → trace the relevant PREMIA logic → provide an evidence-backed explanation → help the support engineer validate the resolution.**

It is **not intended to be a generic chatbot**.

It is a **support investigation platform specialized for PREMIA**.

---

## 2. The Problem Today

A typical production issue can require an engineer to manually go through:

```text
Jira Ticket
     ↓
Understand the issue
     ↓
Identify the PREMIA screen
     ↓
Find the Form
     ↓
Search triggers / PL/SQL
     ↓
Trace dependencies
     ↓
Check database logic
     ↓
Understand the error
     ↓
Determine the root cause
     ↓
Validate the solution
```

### The Challenge

This process can involve **multiple sources of information and significant manual investigation**.

The objective of our platform is to bring this investigation into **one evidence-driven workflow**.

---

## 3. What Will the Product Do?

Think of the product as a **PREMIA Support Engineer's investigation assistant**.

### Input

**A support incident**

Example:

> "Policy number validation is failing while processing the policy."

The system receives the ticket and its available supporting information.

### Processing

```text
        SUPPORT TICKET
              ↓
      Understand Incident
              ↓
       Find Relevant Area
              ↓
      Understand PREMIA Logic
              ↓
       Trace Dependencies
              ↓
      Identify Failure Path
              ↓
       Generate RCA
              ↓
       Validate Finding
```

### Output

The support engineer receives a structured investigation showing:

- **Where the issue is**
- **What part of PREMIA is involved**
- **Why the issue may be occurring**
- **What evidence supports the finding**
- **What should be checked next**
- **Whether the finding has been runtime-confirmed**

---

## 4. Expected Product Experience

### From This

> **"I received a Jira ticket. Now I have to investigate everything manually."**

### To This

> **"I opened the ticket, and the platform has already connected the incident to the relevant PREMIA areas and supporting evidence."**

---

## 5. Expected End-to-End Experience

```text
┌─────────────────────┐
│     JIRA INCIDENT   │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Ticket Understanding│
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ PREMIA Area         │
│ Identification      │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Form / Screen       │
│ Investigation       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Logic & Dependency  │
│ Investigation       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Root Cause Analysis │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Evidence & Validation│
└─────────────────────┘
```

---

## 6. What Makes It Different?

### Evidence-First AI

The platform is designed around a simple principle:

> **AI should reason from evidence, not guess the answer.**

Therefore:

**System evidence → AI reasoning → validation**

rather than:

**Ticket → LLM guess**

This makes the resulting investigation more **explainable, auditable, and useful to support teams**.

---

## 7. Where Are We Now?

### Current Status: Foundation Completed + Jira Intelligence Being Integrated

We have already established the foundation for understanding the PREMIA codebase.

The system has processed approximately:

| PREMIA Intelligence | Current State |
|---|---:|
| Forms | **1,091** |
| Blocks | **4,379** |
| Items | **138,507** |
| Triggers | **57,753** |
| Program Units | **7,688** |
| Dependencies | **244,688** |
| Extracted Errors | **8,101** |

These numbers demonstrate that the platform is no longer just a concept — **the PREMIA codebase has already been processed into a structured intelligence layer.**

---

## 8. What We Have Achieved So Far

### PREMIA Code Understanding

The platform can establish relationships across the PREMIA application:

```text
Forms
  ↓
Screens / Blocks
  ↓
Items
  ↓
Triggers
  ↓
Business Logic
  ↓
Dependencies
  ↓
Database References
```

### RCA Foundation

We have also established the foundation for:

- Identifying relevant code from an incident
- Tracing the relevant logic
- Understanding control flow
- Connecting related dependencies
- Building evidence for RCA
- Using AI reasoning over that evidence
- Performing controlled validation

### Current Focus

We are now connecting this intelligence to **actual Jira support incidents**.

```text
PREMIA Intelligence
        +
Jira Ticket Intelligence
        ↓
Support Investigation
```

---

## 9. Where Are We Going Next?

The product will progressively expand from **code intelligence** into a complete **support investigation workspace**.

### Target Capabilities

```text
Jira Tickets
     │
     ├── Ticket details
     ├── Comments
     └── Attachments
             │
             ▼
      Incident Understanding
             │
             ▼
      PREMIA Intelligence
             │
      ┌──────┼────────┐
      ▼      ▼        ▼
    Forms   Logic   Database
      │      │        │
      └──────┼────────┘
             ▼
            RCA
             │
             ▼
      Runtime Validation
             │
             ▼
       SME Validation
```

---

## 10. Expected Final Outcome

### A Single Support Intelligence Workspace for PREMIA

A support engineer should eventually be able to go from:

> **Jira Incident**

to:

> **Relevant PREMIA area**

to:

> **Relevant Form / screen**

to:

> **Relevant logic and dependencies**

to:

> **Evidence-backed root-cause analysis**

to:

> **Validation and resolution support**

without manually navigating across multiple disconnected sources.

---

## 11. One-Slide Executive Summary

# NIC AI — PREMIA Support Intelligence Platform

### What Are We Building?

An AI-assisted platform that helps PREMIA support engineers **investigate, understand and resolve incidents using evidence from tickets and the PREMIA application landscape.**

### Why?

To reduce the manual effort required to investigate complex PREMIA incidents and make RCA **faster, more explainable and repeatable**.

### Where Are We Now?

**PREMIA intelligence foundation is operational.**

The Forms/source-code landscape has been processed and structured for investigation. We are currently integrating **Jira ticket intelligence** with this foundation.

### Expected Output

```text
Jira Incident
      ↓
Incident Understanding
      ↓
PREMIA Area Identification
      ↓
Code / Dependency Investigation
      ↓
Evidence-backed RCA
      ↓
Validation
      ↓
Support Engineer Decision
```

### Final Vision

> **Transform PREMIA support from manual investigation into an evidence-driven AI-assisted support workflow.**
