# Paper-LLMWiki

A system to use an LLM as a persistent research assistant while writing a scientific paper.  
Inspired by Andrej Karpathy’s *llmwiki* concept, adapted specifically for the scientific-paper workflow.

---

## What this is

- Not note-taking  
- Not one-shot RAG  
- A **compounding knowledge base** that turns experiments, papers, and decisions into a draft

---

## Core idea

Instead of re-deriving your paper every time you write:

`raw experiments + papers → wiki → draft`

The LLM incrementally builds and maintains a structured wiki:

- experiments are summarized once and reused  
- papers are ingested with verified notes  
- decisions and caveats are preserved  

By the time you write the draft, you're stitching together validated pieces — not reconstructing context from scratch.

---

## Why this exists

Writing a paper is mostly bookkeeping:

- What were the exact numbers?  
- Which run did this figure come from?  
- Why did we reject this variant?  

LLMs are good at:

- maintaining structure  
- updating many files consistently  
- tracking cross-references  

This system turns that into an advantage.

---

## How it works in practice

`METHOD.md` is not a one-shot generator.

It turns an LLM into a **stateful maintainer** of your paper:

- you provide raw experiments and papers  
- the LLM incrementally builds and maintains a structured wiki  
- the draft is assembled by quoting and stitching wiki pages  

The key property is persistence: the wiki compounds over time instead of being re-derived at every draft.

A minimal folder structure is required to start (see `template/`).

---

## Get started

1. Copy `template/paper/` into your project  
2. Fill `FOCUS.md` (thesis + claims C1..Cn)  
3. Paste `METHOD.md` into your LLM  
4. Start with: `ingest experiment raw/<your_run>`

## Credit

This project builds on the idea of *llmwiki* by Andrej Karpathy, extending it into a structured workflow for scientific writing, experiment tracking, and citation integrity.
