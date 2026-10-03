---
name: evidence-labels
description: >-
  Use whenever you are about to report that something was checked, tested, verified, or "works" — a handoff, a PR description, a review note, a go/no-go, a "done" message — and especially when some check could NOT run, ran on a subset, or ran before the last edit. Gives every claim one of nine labels (PASS, FAIL, PARTIAL, BLOCKED, INCONCLUSIVE, STALE, ASSUMED, N/A, WAIVED) so a gap in verification can never read as a pass. Symptoms: "tests pass" with a skipped suite, "verified" with no command shown, a green exit from a script that reports gates it did not execute, a claim resting on a check that predates the final edit.
---

# Evidence Labels

## Overview

A verification report fails most often not by lying but by **blurring**: "tests pass" when one
suite was skipped, "verified" when the check ran before the last edit, a green exit from a
script that *reports* gates it did not *execute*. Every one of those reads as PASS to the next
person. This skill replaces the blur with nine labels, each with a fixed meaning, and one rule
for how a gap propagates.

**Core principle: a gap blocks the claim that depends on it, and nothing else.** Partial
verification is reported as partial, never rounded up to complete. Unrelated work is not held
hostage by a gap it does not rest on.

## When to Use

- Any handoff, PR description, review note, or "done" message that says something was checked.
- A script or CI job aggregates several gates and exits 0: label each gate it *ran*, and
  label the ones it only *mentioned*.
- A check could not run (missing tool, missing credential, no network, no cluster).
- A check ran, then you edited the code or its inputs again.
- A reviewer, a subagent, or a local model reports a result you did not observe yourself.

**When NOT to use:** pure prose or design with no checkable claim. (For checking the *content*
of a claim — a number, a count, a filename — use `verify-claims`; this skill is about the
*status* of the check.)

## The nine labels

| Label | Meaning | What it permits |
|---|---|---|
| **PASS** | the check ran and supports the claim | the claim, at the scope the check covered |
| **FAIL** | the check ran and disproves the claim | nothing until fixed; report the output |
| **PARTIAL** | only the named subset was checked | the claim for that subset only — name it |
| **BLOCKED** | a required check could not run; name the missing prerequisite | nothing that rests on it; everything else proceeds |
| **INCONCLUSIVE / CONTESTED** | evidence cannot decide, or sources disagree | nothing; investigate before relying on the claim |
| **STALE** | code or inputs changed after the check ran | nothing; re-run the affected check |
| **ASSUMED** | not checked at all | never sufficient for a correctness or safety claim; say so |
| **N/A** | outside the change's scope, with a reason | skipping it; it is not a substitute for a tool you lacked |
| **WAIVED** | the user explicitly accepted a named gap and its remaining risk | the claim, with the waiver recorded; never permission for an unsafe action |

Three of these are the ones people reach for to avoid saying BLOCKED. Resist:

- **N/A is not "I couldn't run it."** N/A needs an out-of-scope reason. A missing tool is BLOCKED.
- **PASS on a green exit is not PASS on the gate.** An aggregating script's zero exit means the
  script finished. Read what it actually executed; a gate it only printed is ASSUMED.
- **WAIVED is set by the user, in words, for a named gap.** An agent, a loop, or a reviewer never
  sets it. If no one waived it, it stays BLOCKED or ASSUMED.

## The Discipline

1. **List the claims the report makes.** "The fix works", "no regression", "deploys cleanly",
   "the docs are accurate" are four claims; each gets its own label.
2. **For each claim, name the check that supports it** and label the check. Record the
   command, the working directory, the exit status, and the output that matters. One record
   can serve several claims; do not re-run or re-paste it.
3. **Propagate gaps precisely.** A BLOCKED, STALE, INCONCLUSIVE or ASSUMED check blocks every
   claim that rests on it and no other. Say which claims are blocked and which stand.
4. **Re-run what changed.** After the final edit, re-run the checks whose code, configuration,
   or inputs changed. An unrelated prose edit does not invalidate runtime evidence; a one-line
   code edit does. Mark the rest STALE until re-run.
5. **Inspect the assertions, not the count.** Test counts, source-text assertion counts, and
   green exits do not prove coverage. If the claim is "covered", say what the assertion checks.
6. **Report at the scope the evidence supports**, in the same words the labels use. "PASS for
   the unit suite; integration BLOCKED (no cluster); deployment claim therefore BLOCKED" is a
   complete, honest handoff. "All good" is not.

## Choosing checks by what changed

| Change | Sufficient check | Not sufficient |
|---|---|---|
| Docs / policy text | accuracy, references, consistency, and a scenario walk-through for each path the policy names | runtime benchmarks |
| Code | focused tests, affected-component integration, then the regression suites and the lint/type/build checks in scope | "it imported" |
| User-visible behaviour | exercise the real entry point through config, implementation, storage/transport and observable output, including one failure path; state every mock and unwired stub | an in-process unit test alone |
| Deployment / cross-process | build → deploy → drive → observe on the real target, capturing inputs, transitions, persisted state and logs | an exit code; an in-process substitute (that is BLOCKED, not PASS) |

## Handoff template

```
Changed: <files / behaviour>
Checks:
  unit (cd pkg && pytest -q)            PASS   312 passed, exit 0
  integration (make it)                 PARTIAL  only the auth path; billing path not exercised
  lint/type                             PASS
  deploy smoke (minikube)               BLOCKED  no cluster on this machine
  docs references                       PASS   all 6 links resolve
Claims:
  "fix is correct"                      PASS (rests on unit + integration/auth)
  "no regression in billing"            BLOCKED (integration/billing not run)
  "deploy-ready"                        BLOCKED (deploy smoke)
Unresolved risk: <one line each>
User decisions needed: <waive billing gap? / provide cluster?>
```

## Red Flags

- **A report with no labels and no commands.** Every "verified" gets a label and a command.
- **"Tests pass" while a suite was skipped or deselected.** That is PARTIAL; name the subset.
- **A check that predates the last edit.** STALE. Re-run it; the edit is cheap, the blur is not.
- **A result you did not observe** (a subagent said so, a script printed a summary). Label it
  at the scope you can see; ASSUMED until you have the output.
- **Turning BLOCKED into N/A because the tool was missing.** Missing tooling is BLOCKED.
- **Changing a label without re-running the check.** A status changes only because a check
  was re-run and its output supports the new label.

## Pairs with

- `verify-claims` — the content of a claim (numbers, IDs, filenames) against the source.
- `cross-validate` — the independent reviewer gets the labelled evidence, not a summary.
- `unattended-loop` — in an unattended run, BLOCKED / INCONCLUSIVE / CONTESTED are the
  triggers for the blocker procedure, and WAIVED is never set by the loop.
