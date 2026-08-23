---
name: cross-validate
description: >-
  Use when a code change, fix, or written report/analysis is about to be trusted, committed, or delivered and you want an INDEPENDENT reviewer to check it — a fresh Opus-tier Claude reviewer adversarially reviews what you produced, so the two can disagree. Symptoms: "cross-validate this", "get a second opinion on this diff/report", "is this change/finding actually right", before a customer-facing report or a go/no-go, after a non-trivial edit.
---

# Cross-Validate with an Independent Reviewer

## Overview

A model reviewing its own work grades on a curve — it repeats its own blind spots. A **fresh reviewer with no stake in the answer**, prompted to attack rather than agree, gives you disagreement that is signal, not an echo. Use it as the second half of a two-model check: one context produces, another adversarially reviews. Their disagreements are where the bugs and the shaky claims live.

**Core principle: independent disagreement is the product. If the reviewer just agrees, you learned little; if it flags something, investigate before trusting.**

> **Independence caveat.** This skill uses a **fresh Opus-tier Claude subagent** as the reviewer (a clean context and a strong model, prompted to refute). That gives you *context* independence — the reviewer never saw your reasoning, so it can't inherit it — but not *vendor* independence: a same-family model can share the original's blind spots. A truly independent check uses a different vendor's model. Treat a clean bill of health here as "no context-local error found," not "provably correct." When the stakes justify it, add a cross-vendor reviewer on top.
>
> **Reviewer tier: Opus, not the absolute ceiling.** Route the reviewer to **Opus** (the standard high-capability tier) — not to a still-larger model like Fable by default. Opus is the deliberate cap for routine cross-validation: it is a strong, independent reviewer without the cost/latency of escalating every check to the maximum tier. Reserve a larger model for the rare review the user explicitly asks to escalate, or a genuinely load-bearing go/no-go.

**The reviewer points at problems; it does not hand you solutions.** Use it as a *problem-finder* only. Take its FINDING (the "something is wrong here") as a lead to verify — never take its FIX. A reviewer that spots a real bug will often propose a wrong or inverted repair (it reasons about the symptom, not your system's constraints). Once you've confirmed a finding is real, **derive the fix yourself from your own analysis of the design** — do not port the reviewer's suggested code, boundary, value, or approach. Verifying a finding is not verifying its fix; they are separate acts and the second is yours.

> Worked example (real): an independent reviewer correctly found a size-binning routine misrouted low-density content (a genuine bug the local invariant test missed). Its proposed fix — flip the ratio to the upper bound — made the *unsafe* misroute systematic (the economics made the opposite direction the dangerous one). Finding: REAL. Fix: REJECTED. The right repair (delete the ratio entirely and share one observable across both planes) came from analyzing the system, not from the reviewer. Taking its fix would have shipped a worse bug with an independent-review stamp on it.

**Related:** `verify-claims` checks a report's *numbers* against source data; this checks a change's *code/reasoning* against an independent reviewer. A customer-facing report deserves both.

## When to Use

- After a non-trivial code edit, before committing or calling it done.
- Before a report / analysis / go-no-go goes to a customer or drives a decision.
- When a finding "feels right" but nothing external has challenged it.

**When NOT to use:** trivial mechanical edits; or when the thing to check is a bare number from an audited pipeline (that's `verify-claims`).

## How to Invoke the Reviewer (read-only — it must not mutate what it reviews)

Spawn the reviewer as a **subagent** with a fresh context, on the **Opus tier** (escalate to Opus if the work was produced on a smaller model; stay on Opus if it wasn't — do **not** jump to a larger model like Fable by default), and give it **read-only** tools so a reviewer can't edit the code under review.

The subagent's final report is the review — you read it and reconcile it yourself; it is not shown to the user directly.

**Prompt shape (code change in a git repo):**
```
You are an adversarial code reviewer. Review ONLY the diff below (or: the diff of
<branch> vs <base>). Your job is to REFUTE it — find correctness bugs, broken edge
cases, and unsupported claims. Be specific: cite file:line for each finding. State
a concrete failure scenario (inputs → wrong output) per finding. Do NOT propose
fixes, and do NOT just agree — if you find nothing, say so and say why the code
holds. DIFF:
<paste `git diff <base>...<branch>` or the uncommitted diff>
```

**Prompt shape (files / a written report):** paste the file(s) or report and ask the reviewer to challenge the *reasoning and every quantitative claim*, not to praise it. For a report, tell it explicitly: "refute the reasoning and check every number against what's shown."

Give the reviewer the **artifact itself** (the diff, the files, the report) — never your summary of it — so it forms an independent view rather than critiquing your framing. Read its findings; don't relay its verdict unread — a subagent reviewer hallucinates too. Its finding is a *lead* to verify.

### Invoke in Pi and Claude CLI

- **Claude Code:** spawn the reviewer as an Opus-tier **subagent** with read-only tools
  (above), or force the skill with `/apex-workflow:cross-validate`.
- **Pi:** use a per-task cue to run the review on an independent higher tier without
  leaving your session — `>>deep <paste diff/report + refute prompt>` (Opus) or
  `>>kimi …` (a different *family*, the strongest independence). The cue restores your
  model afterward. Force the skill itself with `/skill:cross-validate`.
- **Either:** for a big change, chain it — in Pi, `/learn` already ends on a heavy tier;
  for code, hand the diff to `>>deep`/`>>kimi` and read the refutation before you commit.

### Multi-model reconciliation (design → review → reconcile)

For a substantial design or build, one review pass isn't the whole loop. Escalate to a
**reconciliation chain across *families*** so no single model's blind spots survive:

1. **Design/produce** on a heavy tier.
2. **Independent review** on a *different family* (e.g. produce on Claude, review on Kimi —
   cross-*family* disagreement is stronger than same-family).
3. **Reconcile:** feed the review + the fixes back to the original reviewer for a final
   pass; ship only when it confirms the fixes and finds no new blocker.

The disagreements between families are the product. A defect that both a Claude reviewer
and a Kimi reviewer miss is rare; a defect one catches is exactly what this loop is for.

## The Discipline

1. **Give the reviewer the same source of truth, not your summary.** Point it at the diff / files / report, so it forms an independent view — not a critique of your description.
2. **Prompt it to REFUTE, not confirm.** "Find what's wrong / unsupported / missing," never "does this look good?" A confirm-prompt gets you an echo. Ask it to *locate problems* — not to propose fixes; you won't use them anyway.
3. **Read the findings and triage each FINDING** (not its fix): CONFIRMED (the problem is real — verified at ground truth), REJECTED (the reviewer is wrong — say why), UNCERTAIN (dig in). You are not obligated to accept; you ARE obligated to resolve. Watch for findings that wander outside the artifact you gave it (a reviewer handed a diff will sometimes flag pre-existing code it happened to see) — those are out of scope for this change.
4. **On disagreement, go to ground truth.** Don't average two model opinions — run the test, grep the data, check the oracle. The disagreement told you *where* to look.
5. **Fix confirmed findings YOURSELF, from your own analysis.** Ignore whatever remedy the reviewer offered. Design the repair against your system's actual constraints, then verify *that* independently (its own test/ground-truth check) — a confirmed finding does not make the reviewer's fix correct. If your analysis of the fix keeps not converging, that's a signal about the design (maybe the construct should be removed), not a cue to reach for the reviewer's suggestion.
6. **Report the reconciliation, not just "the reviewer agreed."** State what was flagged, whether the *finding* survived, and what *you* changed and why (including where you rejected the reviewer's proposed fix).
7. **Cap the loop at 2 passes — an independent review is expensive, so spend it deliberately.** A fix is a *change*, and a change can introduce a new defect, so the original review plus one validation of its fixes is the standard budget: **pass 1** reviews the work; **pass 2** (only if pass 1 produced non-trivial fixes) validates *those fixes* (spawn a fresh reviewer — don't continue the same one). Stop after pass 2. If pass 2 still leaves fixes you can't verify by inspection, verify at ground truth yourself (run the test, grep the oracle), don't burn another review. A third pass happens only if the user explicitly asks for it. Prefer to terminate at pass 1 outright when its fixes are trivially verifiable by inspection — small, local, each with a test or a direct check you can eyeball; spend pass 2 only when the fixes are large, cross-cutting, or subtle. (Worked example: on one batch, pass two caught a regression that pass one's own fix had introduced — routing logic the first fix rewrote dropped a field. That is exactly what the one allowed re-run is for.)

## Third check: the codeqa grounding oracle (facts, not a second vote)

The independent reviewer is a second *reasoner*. codeqa adds a different KIND of check: a
**deterministic ground-truth oracle** for the `file:line` citations in a finding or report. It is
**not another opinion to average** (the discipline forbids averaging) — it produces a *fact*: does the
cited code actually exist, and does the cited span fall within the live file? That is exactly the "go
to ground truth on disagreement" step (Discipline #4), automated. Run it on **every** cross-validate
pass whose artifact cites code; it self-skips when nothing is groundable.

**Run it** (reads the finding/report from stdin or a file; `CODEQA_REPOS` points at your repo configs):
```bash
echo "<the finding text, incl. its file:line citations>" \
  | CODEQA_REPOS="$CODEQA_REPOS" python -m apex_router.codeqa.cli ground --check
# --check -> exit 2 if any citation is stale; add --json for a machine-readable verdict.
```

**Verdicts** (per citation, against your registered repos):
- **grounded** — the file exists and the cited span is within it. The citation is real.
- **stale** — the file exists but the span runs past end-of-file (moved/shrunk). A REAL defect
  (`--check` → exit 2): the finding cites a line that isn't there.
- **unverified** — *advisory only.* The path is name-owned by a registered repo but can't be
  positively located there — it may be a real file in an UNREGISTERED sibling repo (e.g. a `foo/…`
  path that actually lives in a `foo-ext` sibling), an unreadable file, or an invented path. A
  deterministic filesystem oracle can't tell these apart, so it refuses to accuse. **Never reject a
  finding on `unverified`.**
- **not applicable** — the artifact cites no groundable code; the oracle stays silent (honest skip).

Scope honesty: this oracle deliberately does NOT emit a "hallucinated" verdict. Distinguishing an
invented citation from a real file in an unregistered sibling repo is beyond a filesystem check
(every heuristic that tried false-accused real code) — a false accusation would sink a VALID finding,
worse than a missed catch. The invented-vs-real call is left to the model-side answer-verification
gate (which sees the retrieved chunks) or to `verify-claims`.

**How to use the verdict:** a `stale` (exit 2) means the finding cites a line that isn't there — treat
that finding as **factually broken regardless of how plausible its prose is**. `grounded` does NOT
vindicate a finding — it only proves the cited *location* is real, not that the reasoning about it is
correct; the independent reviewer still owns the reasoning review. An `unverified` is a nudge to check
that citation by hand, not a reason to reject. Doctrine unchanged: don't average opinions — the oracle
hands you a fact (grounded/stale) to act on, not a vote to tally.

## Common Mistakes

- **Prompting for approval.** "Does this look correct?" yields agreement. Prompt to refute.
- **Taking the reviewer's FIX, not just its finding.** The single biggest trap: the reviewer spots a real bug, proposes a repair, and the repair is wrong/inverted. Adopt the *finding*, author the *fix* yourself. A confirmed finding is not a validated fix.
- **Relaying the reviewer's output unread** as if it were truth — it hallucinates too. Its finding is a *lead* to verify, same discipline as any model claim.
- **Averaging the two opinions.** Two models disagreeing is not "50% confidence" — it's a specific place to run a test or check the data.
- **Letting the reviewer write while reviewing.** Give it read-only tools; a reviewer that can edit the code isn't an independent check.
- **Feeding it your summary instead of the artifact.** Then it reviews your framing, not the work.
- **Treating a same-tier reviewer as fully independent.** Reviewing with the same (or a weaker) model shares blind spots. Escalate to Opus (the reviewer cap), and for high stakes add a cross-vendor check. Don't reflexively escalate past Opus to the largest model — that's for explicit escalations, not every review.

## Red Flags — STOP

- About to commit / deliver a non-trivial change or a customer-facing report with only one context's eyes on it.
- The reviewer flagged something and you're about to dismiss it without checking the source.
- You're about to paste/adapt the reviewer's suggested fix. Stop — confirm the finding, then design your own repair. Its fix can be wrong even when its finding is right.
- "The reviewer agreed, so it's fine" — agreement from a confirm-prompt proves nothing; verify it saw the real artifact and was asked to refute.

**All of these mean: spawn the independent reviewer read-only on a higher tier, prompt it to refute, and reconcile every flag against ground truth before trusting the result.**
