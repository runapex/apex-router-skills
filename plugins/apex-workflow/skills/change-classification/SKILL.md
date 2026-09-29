---
name: change-classification
description: >-
  Use when you want a per-change risk read from MORE THAN ONE independent model — after a commit or an autonomous-loop step, before trusting a diff, when you need to see where reviewers AGREE (trust) vs DISAGREE (look here). A panel of independent models classifies the change on three axes — change class, requirement fit, blast-radius damage — from the diff + the test/CI output + the requirements, and reports their divergence. Symptoms: "classify this change", "how risky is this diff", "get a multi-model read", "where do the reviewers disagree", "richer signal each loop", measuring change quality across a series of commits.
---

# Change Classification — a multi-model risk read per change

## Overview

A single reviewer gives you one opinion; you can't tell a confident-right from a
confident-wrong. A **panel of independent models**, each answering the SAME fixed
schema about the SAME change, gives you something a single opinion can't:
**agreement is trust, and disagreement is a map.** Where the panel converges, the
read has supporting evidence, not proof of safety. Where it splits — one says blast is `low`, another `high`
— that axis is exactly where to look before you trust the change.

**Core principle: the product is the DIVERGENCE, not the labels.** This is a
*measurement* tool, never a gate. It does not pass or fail a change; it tells you
*which axis is contested* so your own judgment (or a `cross-validate` pass) goes
straight to the risk instead of re-reading the whole diff.

It composes with the rest of the loop: `disciplined-execution` runs the change,
`cross-validate` gets an adversarial review of the fix, `verify-claims` checks any
number — and this skill, run each step, shows you where the change's *risk*
concentrated so you spend the expensive review where the panel says it matters.

## When to Use

- **Per step of an autonomous develop→test loop** — classify each change so you get
  a richer signal than "tests passed": *what kind* of change, *does it fit the
  requirement*, *what's the blast radius*, and *where do independent models
  disagree*.
- **Before trusting a non-trivial diff** — a HIGH-blast rating or a requirement-fit
  disagreement is a cue to slow down and cross-validate that specific axis.
- **Across a series of commits** — the divergence flags accumulate into a map of
  which changes were contested and on which axis.

**When NOT to use:** a trivial mechanical edit (rename, formatting); or when you
only need one adversarial review of correctness (that's `cross-validate`), or only
a number checked (that's `verify-claims`). This is for *multi-model dynamics on a
change*, not a single yes/no.

## The three axes (one fixed schema, so answers are comparable)

Every panel member returns exactly this, which is what makes divergence measurable:

- **change class** — `schema-migration | data-model | compute-core | validation |
  api-surface | search-index | config-gating | test | docs | mixed`. (A model may
  coin its own — that's signal too.)
- **requirement fit** — `advances | partial | off-target | regresses-req |
  neutral`, plus which requirements/ACs it touched and why. Judged against the
  *requirements you supply*, not the model's guess at intent.
- **blast-radius damage** — `none | low | medium | high | critical`, the surfaces
  at risk, and a **regression signal** read from the test/CI output you supply
  (`confirmed | refuted | not-covered`) — so a model can't claim "safe" when
  nothing actually exercised the change.

## How to run it

The tool is `change_classifier.py` in apex-router's `scripts/`. It takes the
**change** (a diff, or a repo+base to diff), the **regression signal** (a test/CI
log), and the **requirements** (one or more spec files), runs the panel in
parallel, and prints the per-model classifications plus the divergence summary.

```bash
python3 scripts/change_classifier.py \
  --repo . --base HEAD \
  --tests test_run.log \
  --reqs spec.md,acceptance.md \
  --panel panel.json \
  --out report.json
```

Use `--diff change.diff` (or `-` for stdin) instead of `--repo` for a curated diff.
`--repo` includes tracked changes only; add `--include-untracked` explicitly to
include untracked files. Review inputs for secrets before giving them to a hosted
panel. `--tests` supplies the regression signal; omit it only when none exists.
`--reqs` is a comma-separated list. The full per-model report goes to `--out`;
stdout contains the divergence summary and each member's success status. Large
inputs are clipped; `input_clipped_chars` in the report and an `INCOMPLETE INPUT`
flag identify omitted material. Such a report is not a review of the whole change.
Treat reports as sensitive: failed commands' diagnostic tails can contain private
data. Inputs are untrusted; panel agreement is not protection from prompt injection.

### The panel is config-driven — bring your own models

A panel member is just a **name** and an **argv command** that reads the prompt on
**stdin** and prints the model's answer (containing one JSON object) on **stdout**.
Anything that satisfies that is a valid member — a hosted-model CLI, a local
OpenAI-compatible server behind a three-line wrapper, or a shell function:

```json
[
  {"name": "reviewer-a", "cmd": ["your-model-cli", "--model", "<id-a>", "--stdin"]},
  {"name": "reviewer-b", "cmd": ["your-model-cli", "--model", "<id-b>", "--stdin"]}
]
```

Point the tool at it with `--panel panel.json` or the `CLASSIFIER_PANEL` env var.
With neither, the tool prints this example and exits — it **never hardcodes a model
id**, so nothing about your setup ships with it.

## Reading the result — the discipline

1. **Look at the flags first.** The divergence block ends in `flags`. `panel
   broadly agrees` means the measured axes converge, not that the change is safe.
   Anything else names the axis
   that split — a `requirement-fit DISAGREEMENT`, a `blast-radius SPREAD>=2`, a
   `regression-signal DISAGREEMENT`, "at least one model rates blast HIGH", "at
   least one model says the change REGRESSES a requirement."
2. **Go to the contested axis, not the whole diff.** The point of the panel is to
   aim your attention. If blast split low/high, read *why* each said so (the
   per-model `blast_radius.why`) and settle it at ground truth.
3. **Treat a HIGH/CRITICAL or a `regresses-req` as a cue to `cross-validate`** that
   specific concern — this skill *finds* the risk; the fix and the deeper review
   are separate acts you own.
4. **Trust convergence, but not blindly.** A panel that agrees can still be
   agreeing wrongly if its members share a family/blind spot (see below). Agreement
   lowers the odds of a context-local miss; it is not proof.
5. **Never let the tool decide.** It emits no pass/fail and takes no action.
   Regression signal `not-covered` across the panel is itself the finding: the
   change ran but nothing tested it.

## Make the panel genuinely independent

The whole value is that members **fail differently**. Two members of the same
model family share blind spots, so their agreement is weak evidence. Span
**different families/vendors** where you can. Two-to-three members is the sweet
spot: enough to expose divergence, cheap enough to run on every change. Raise the
panel's capability tier for a high-risk change (a core-logic or migration diff),
drop it for routine ones — the same judgment `model-routing` applies elsewhere.

## Failure modes (fail loud, not silent)

- **A member's command fails to start, exits nonzero, or times out** → that member is marked
  `ok:false` with the reason; the panel continues with the rest. A one-member panel
  can't diverge — the tool says so (`need >=2 valid classifications`). If every
  member fails, the report is still emitted but the CLI exits 1 (operational
  failure, not a risk gate). High-risk classifications do not change the exit code.
- **A model wraps its JSON in prose or emits a decoy sub-object** → the extractor
  selects the object carrying the required top-level keys, not a trailing fragment.
  JSON strings may contain braces. Only complete, structurally valid classifications
  on stdout count; error objects, partial answers, and stderr do not. Invalid output
  is `ok:false` (inspect its `raw_tail`), not silently counted as agreement.
- **No test output supplied** → regression signal will read `not-covered`; that's
  honest, not a pass. Supply the real test/CI log to get a grounded blast read.
- **Panel agrees but all rate the change unsafe** → that's not divergence, it's a
  consensus finding — act on it.

## Red Flags — STOP

- You're about to trust a diff the panel split on (blast low vs high, or a
  requirement-fit disagreement) **without** reading the contested axis.
- You're treating the classifier's labels as a gate ("it said low, ship it") — it
  is a measurement, not a decision.
- Your panel is two members of the **same** model family and you're reading their
  agreement as strong evidence — it isn't; add a different family.
- The panel says `regression_signal: not-covered` and you're reading a `low` blast
  as "safe" — nothing tested it; that's unknown, not safe.

**All of these mean: run the panel on the change, read the divergence flags first,
take your judgment to the axis the models disagree on, and let the split — not the
label — tell you where the risk is.**
