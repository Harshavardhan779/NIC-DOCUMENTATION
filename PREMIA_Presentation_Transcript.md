# PREMIA Support Intelligence: Presentation Transcript

Total speaking time: about 9 to 10 minutes across 8 slides. Written to be spoken, so adjust the wording to your own voice.

## Slide 1: The problem (about 1 min)

Let me start with the problem.

When a production issue comes in on PREMIA, it arrives as a Jira ticket. From there, the support engineer has to investigate almost entirely by hand. First they understand the issue. Then they work out which PREMIA screen is involved and find the Form behind it. Then they search through triggers and PL/SQL, trace the dependencies, check the database logic, and only after all of that can they determine the root cause.

The important point is that every one of those steps lives in a different place: the ticket, the Forms, the code, the database logic. And each step depends on manual effort. So investigations are slow, and they are hard to make consistent and repeatable.

That is the problem we are solving: bringing this whole investigation into one evidence-driven workflow.

## Slide 2: The solution (about 1 min)

So what are we building? A PREMIA-specific investigation assistant. To be clear, this is not a generic chatbot. It is built for PREMIA support.

The flow follows how an engineer already thinks. We start with the support ticket and understand the incident. We identify the relevant area of PREMIA, then trace the logic and dependencies. From that we build a root cause analysis, and then we validate it.

What the engineer gets at the end is a structured investigation. It shows where the issue is and which part of PREMIA is involved. It explains why it may be happening, along with the evidence behind that explanation. And it tells them what to check next, and whether the finding has been confirmed at runtime.

## Slide 3: Our approach (about 1.5 min)

The principle behind all of this is: evidence first.

A simple approach would be to hand the ticket to an LLM and ask for an answer. That gives you a guess that is hard to explain and hard to audit. Our approach is different. The system gathers evidence from the PREMIA code, the AI reasons over that evidence, and then the result is validated. That makes it explainable, auditable and repeatable.

We also grade every finding. It starts as a hypothesis. It becomes static evidence when it is backed by the source. It becomes runtime confirmed when it is checked against the running system. And it becomes SME validated when a subject matter expert signs it off. A finding is never silently upgraded from one level to the next, and the final authority is always the SME.

## Slide 4: Where we are (about 1 min)

Now, where are we today? The foundation is already in place.

The PREMIA codebase has been processed into a structured intelligence layer. That is 1,091 Forms, about 57,000 triggers, about 138,000 items and about 245,000 dependencies. Beyond that, we have mapped around 56,000 control-flow graphs, 7,688 program units and 8,101 indexed errors.

So this is no longer a concept. The PREMIA code is already structured and searchable. What we are doing now is connecting Jira ticket intelligence to this foundation.

## Slide 5: Available today (about 1.5 min)

Let me walk through what the platform can already do.

On the ingestion side, it takes in project dumps and keeps an artifact registry. It converts Oracle Forms FMB files to XML, incrementally, so we do not redo work. It then builds a structured index of Forms, dependencies and errors, with full-text search.

On the analysis side, it does deterministic control-flow analysis, so we can follow how the logic actually runs. It retrieves evidence for an RCA, along with metadata on how good that retrieval is. A sufficiency gate then checks that the evidence is enough before an RCA is attempted. And only a bounded, relevant amount of context goes to the LLM.

The LLM layer uses an OpenAI-compatible provider and validates every response strictly. We also have the groundwork for controlled runtime confirmation. All of this is backed by an extensive automated regression test suite.

## Slide 6: Why this approach (about 1.5 min)

Why did we choose this architecture? Six reasons.

First, trustworthy answers. Deterministic analysis is the source of truth. The LLM only proposes hypotheses from evidence, so it cannot invent facts about the source code.

Second, it is reproducible. Every investigation is tied to a fixed snapshot of the source, the tooling and the model version, so we can replay it later.

Third, runtime checks are safe. They use approved templates only, are read-only, need human approval, run in isolation, and are fully audited.

Fourth, customer isolation. Each customer gets their own deployment cell, which keeps their source code separate and protected.

Fifth, it is simple to run. It is a modular monolith using SQLite and PostgreSQL, with no heavy infrastructure at our current scale.

And sixth, delivery is measured and low risk. We build on the validated proof of concept, each phase needs human sign-off, and we track quality against SME-validated cases.

## Slide 7: Roadmap (about 1.5 min)

Here is how we get from here to the full workspace.

Right now, we are strengthening the core: making sure artifact identity is stable and consistent, so the same source always produces the same result.

Next, we build the investigation model and prove it with a first real ticket, taken through to an RCA and an SME check. In parallel comes Jira ticket intelligence: tickets, comments, attachments and OCR.

Then we add the evidence ledger with citation checks, improve retrieval, run an experiment on whether a bounded agent actually helps, and bring in controlled runtime evidence.

Later, we add the Jira workspace with write-back, the full UI and explorers, PREMIA domain knowledge, and production hardening.

The Jira integration and the UI are still ahead of us, and we will only adopt the agent if the experiment shows a measurable benefit.

## Slide 8: Summary (about 30 sec)

To sum up: our goal is to turn PREMIA support from manual investigation into an evidence-driven, AI-assisted workflow.

An incident comes in from Jira, we identify the PREMIA area, trace the logic and dependencies, produce an evidence-backed RCA, and validate it. The foundation is already built, and we are now connecting it to real support tickets.

Thank you. I am happy to take questions.

## Likely questions

**Is this a chatbot?**  
No. It is an investigation platform built specifically for PREMIA. It works from evidence in the PREMIA code, not from general knowledge.

**Does the AI decide the root cause?**  
No. The AI proposes hypotheses and reasons over evidence. Findings are graded, and the SME has the final say.

**Can it run queries on our database?**  
Only in a very controlled way: approved templates, read-only access, human approval, isolation and audit. No free-form SQL reaches the database.

**Is Jira integrated yet?**  
Not yet. The PREMIA code intelligence is in place and Jira ticket intelligence is the next step.

**How do we know an RCA is right?**  
We check that every citation points to real evidence, confirm at runtime where it applies, and have an SME validate it. Validated cases also build a golden dataset to measure quality over time.
