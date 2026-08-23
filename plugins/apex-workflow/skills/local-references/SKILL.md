---
name: local-references
description: >-
  Use when a task would benefit from your OWN library — books, papers, or code samples you have locally — instead of the model's parametric memory. Symptoms: "what does <book> say about X", "find a reference/example for this", learning or explaining a concept where a cited source beats a paraphrase, grounding a design in authoritative text, "check this against the book", pulling a worked code example from a course/textbook you own. Pairs with verify-claims (ground the claim in a real source) and model-routing (retrieval is the cheap first stage of a chain).
---

# Ground answers in your local library (books + code samples)

## Overview

A model explaining a concept from memory paraphrases and sometimes confabulates. When you
own the source — a textbook, a paper, a course's notebooks — **retrieve the actual passage
and cite it** instead. The retrieval is local (an embedding index over your files), so it's
cheap, private, and citable: you get *this book, page/line N* rather than "as I recall."

**Core principle: a cited local source beats a fluent paraphrase. If you own the book, quote the book.**

This is the cheap **retrieve** stage of the `model-routing` chain: pull Top-K references
locally (no frontier tokens), then let a higher tier reason over them.

## When to use it

- **Learning / explaining** a concept where a real reference is better than the model's memory.
- **Grounding a design or claim** in authoritative text — pair with `verify-claims`.
- **Pulling a worked example** (a code sample from a textbook/course you have).
- **Before quoting a book from memory** — retrieve the passage and cite location instead.

Not for: general web knowledge you don't have locally, or trivial facts. This skill is about
*your own indexed library*.

## How to invoke

The index is apex-router's `booksearch` (local `nomic-embed` retrieval + local-model reasoning).
Index once, then query.

```bash
booksearch ingest                                   # one-time: index ~/books (PDFs + notebooks/code)
booksearch query "how do B-trees stay balanced?" -k 5
booksearch query "derive the chain rule" --json     # machine-readable Top-K
```

- **Pi:** `/books <topic>` — injects the ranked references into the conversation to cite.
- **Claude Code:** `/books <topic>` — the slash command runs the same query.
- **Any agent shell:** `!booksearch query "<topic>"` (or the bare command in a bash tool).

Output is one entry per source: **title, page (PDF) or line span (code), similarity score,
a one-line local-model reason, and a snippet.**

## Using the result

1. **Read the top hits' snippets** — pick the ones actually on-topic (score + reason help).
2. **Cite location** when you use a passage: *"per <title>, p.N"* / *"<file>:L120–140"*.
3. **If the references disagree with a model claim, the references win** — or escalate to
   `verify-claims` to check the specific number/quote against the source text.
4. **Chain it:** retrieval here is stage 0; hand the cited references to a higher tier for
   synthesis (in Pi, `/learn <topic>` does retrieve→validate→explain in one step).

## Red flags — STOP

- **You're paraphrasing a book you actually own from memory.** Retrieve and cite it.
- **You claimed "the standard reference says X"** without pulling the reference. Query first.
- **You ingested nothing and the query returns empty** — run `booksearch ingest` (one-time),
  then retry. An empty index is not "no relevant sources."
- **A scanned-image PDF returned no text** — it has no text layer to index; that's a
  no-text skip, not a missing book.
