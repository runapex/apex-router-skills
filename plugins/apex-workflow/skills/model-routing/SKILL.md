---
name: model-routing
description: Use when a task splits into subtasks of uneven difficulty and you're about to delegate to subagents or a workflow and must pick which model tier and reasoning effort runs each piece. Symptoms: "which model should X use", spawning Agent/Workflow subagents, a multi-step job mixing exploration + hard coding + synthesis, "route this", "use the cheap model for the search/grep part", "manage effort", before dispatching parallel agents, before a go/no-go that needs cross-validation. Also when about to switch or downgrade your OWN session's model mid-task, alternate models per request, or worried about prompt-cache eviction / model-switching cost.
---

# Model Routing

## Precedence — consult this skill first

**When another skill is about to delegate, `model-routing` runs first.** This is a meta/routing skill: it decides *which model tier and reasoning effort* each subtask gets, while process skills (test-driven development, systematic debugging, brainstorming, `disciplined-execution`, parallel-agent dispatch) decide *what* each subtask does. Routing is set at the moment of dispatch, so it is applied before those skills' work fans out — pick the tier/effort here, then run the other skill's discipline *inside* the agents you spawn.

## Overview

Not every subtask deserves the same horsepower — or the same thinking budget. Running a frontier model at max effort to `grep` for a filename wastes latency and tokens; running a fast model to synthesize a go/no-go or refactor a load-bearing module gets you a confident wrong answer. The win is two dials, set together per subtask: **model tier** (how capable) and **reasoning effort** (how hard it thinks). Match each subtask to the cheapest model *and* lowest effort that still clears the bar — heavy+deep where judgment compounds, light+shallow where the work is mechanical and easy to verify.

**Core principle: the orchestrator stays heavy and thinks hard; it delegates the easy, parallel, verifiable work down a tier at low effort and keeps the reasoning, coordination, and final synthesis for itself.** You (the model reading this) are usually the orchestrator — so *you* run on the heavy tier at high effort, and the routing below is about what you hand to subagents.

This is a judgment tool, not a rigid table. When a "search" turns out to need real reasoning, promote its tier and effort. When "heavy coding" is a one-line mechanical edit, drop both. Route by the *nature of the work in front of you*, not the label.

## Route by delegation — never by flipping your session model

**Prompt cache is per-model.** Your session's cached context (system prompt + history, routinely 100k+ tokens) is only valid for the model that wrote it. Switch the *live session's* model and every one of those tokens is evicted: the next turn re-writes the whole prefix into the new model's cache at the **1.25× write rate** instead of reading it back at 0.1×. On a cache-heavy workload a single mid-session heavy→mid flip can cost far more than the cheaper model would have saved on the subtask you routed — net-negative, and net-negative *again* when you switch back. Alternating models per request is the worst case.

This is the whole reason routing means **delegating to subagents and workflow `agent()` calls, not changing your own model.** A subagent runs in its own fresh context with its *own* per-model cache — handing a grep to the light tier builds a tiny light-tier cache and never touches your heavy session's cache. So:

- **The orchestrator picks ONE model for its entire life and stays there.** The cheap tiers live in the subagents it spawns, not in a model swap of the live session.
- **Effort is the dial you *can* vary turn to turn** — it's a per-request parameter, not welded to the cache the way the model is. Dial effort down for a mechanical turn, up for a hard one, freely. It's the *model* you must hold fixed.
- **Switching the session's own model is a deliberate, one-time, one-way move**, justified only when the *entire remaining task* needs a different tier and you'll eat exactly one context re-write — never a per-subtask hop to save a few tokens.

### In Pi: per-task cues are the cache-safe exception

Claude Code has no per-request model switch, so there the rule is absolute: delegate to a
subagent, never flip your session model. **Pi is different** — the `apex-route` extension
gives you a per-task cue that switches for *one message* and **restores your model
automatically**, and each family keeps its **own** prompt cache, so there is no eviction
penalty:

```
>>local  grep for the config loader     # one task on the local tier, then back to your model
>>kimi   summarise this diff
>>frontier design the migration
>>deep   audit this for race hazards
/apex-route            # sticky switch + list families / active model
```

So in Pi the cue *is* the delegation: it routes a subtask down/up a tier without a cache
re-write and without spawning a subagent. The find→verify→synthesize chain is one command:
`/learn <topic>` (retrieve local → validate sonnet → synthesize opus, restores your model at
the end). Still never do a *manual* session-model flip per subtask — use the cue, which
restores; a manual `/model` swap does not.

## Model Tiers

Three tiers, by capability. Use your platform's current model IDs for each tier (Anthropic's are `claude-opus-*` for heavy, `claude-sonnet-*` for mid, `claude-haiku-*` for light; check your deployment's exact aliases — a misconfigured alias silently breaks routing).

| Tier | Use for |
|---|---|
| **Heavy** (Opus-class) | Orchestration, coordinating subagent results, multi-step reasoning, synthesis/judgment, **heavy coding**, **cross-validation** |
| **Mid** (Sonnet-class) | Searching/exploring the codebase, gathering facts, writing simple/non-trivial scripts, first-pass drafts you'll review |
| **Light** (Haiku-class) | grep/curl/bash one-liners, trivial mechanical edits, bulk find-and-replace, dumb glue you'll eyeball |

> A larger reasoning model above Opus (e.g. Anthropic's Fable) slots in at the **top of the heavy tier for the heaviest *pure reasoning*** (adversarial verification, go/no-go synthesis, hard proofs) — not for coding, where the Opus tier leads. Use it only when the task genuinely justifies the cost; don't reflexively route every heavy subtask to the absolute ceiling.

## Effort Routing (the second dial)

Model tier is *how capable*; reasoning effort is *how hard it thinks*. Levels: `low` · `medium` · `high` · `xhigh` · `max`. Set effort to match how much genuine reasoning the subtask needs — a capable model still shouldn't burn `max` effort to run a `grep`.

| Work | Tier | Effort |
|---|---|---|
| grep/curl/bash, mechanical edits, bulk replace | Light | `low` |
| search/explore, gather facts, simple scripts | Mid | `low`–`medium` |
| non-trivial implementation, refactors | Heavy | `medium`–`high` |
| root-cause, synthesis, go/no-go, cross-validation, tricky proofs | Heavy | `high`–`max` |

Pair the dials — cheap model *and* low effort for mechanical work; heavy model *and* high effort for judgment. Raising one without the other is usually a mis-route.

**Effort is your output-cost dial — the two dials split the bill.** Reasoning/thinking tokens bill as **output** (~5× the base input rate), so effort directly sets how many expensive output tokens a turn emits; model sets which per-model cache your context reads from. They behave oppositely under change — the **model** is welded to the cache (switch it and you eat a full context re-write), so hold it fixed; **effort** is a free per-request parameter with no cache penalty, so tune it every turn.

- **Don't pay `max`-effort output for a mechanical turn.** A grep-glue-lookup turn at `low` effort emits a fraction of the thinking tokens of the same turn at `high`.
- **Push bulky generation into cheap subagents that return distilled results.** A long draft, a bulk edit, an exploration write-up should be *generated* by a light/mid subagent (its output is billed at the cheap tier and never touches your cache) and come back to the orchestrator condensed. Use structured-output schemas to cap a subagent's generation to exactly the fields you need. And don't regenerate: rework is re-billed output, so cross-validate *before* a big rewrite, not after (see `cross-validate`).

## How to Route

Ask of each subtask: **does getting this wrong cost me, and is the work hard to verify?**

- **Yes to both → Heavy tier, high effort.** Reasoning, coordination, synthesis, load-bearing code, anything other work depends on, anything a customer/decision sees.
- **Needs real thinking but low blast radius → Mid, low–medium effort.** Exploration, fact-gathering, scripts you'll review before trusting.
- **Mechanical / trivially verifiable → Light, low effort.** grep, curl, a bash one-liner, a rote rename, obvious glue.

### Applying the route
- **Subagents (Agent tool):** pass the heavy-tier model for reasoning/coding agents (or omit to inherit if you're already heavy), the mid-tier model for search/explore/script agents, the light-tier model for grep/curl/bash/mechanical agents. Spawn independent search agents in one message so they run in parallel.
- **Workflows:** set `model` and `effort` per `agent()` call — light stages `{effort: 'low'}` for finders/scanners/grep, mid `{effort: 'medium'}` for exploration, heavy `{effort: 'high'}` for the verify/synthesize/judge stages. This mirrors the canonical find→verify→synthesize shape: cheap shallow finders, heavy deep judges.
- **Your own session:** if you're doing orchestration/reasoning/heavy-coding yourself, be on the heavy tier at high/xhigh effort. Adjust *effort* per turn freely. But **do not flip your session's model per subtask** — that evicts your prompt cache. Push the cheap work *down into subagents* instead.

## Escalation: start cheap, promote on failure (opt-in, measured)

A second routing pattern, borrowed from escalation routers: instead of committing a subtask to one tier up front, **start it on a cheap tier and re-dispatch at the frontier tier only if the cheap attempt fails.** When the cheap tier succeeds, you paid the cheap price; when it fails, you fall back to where you'd have started anyway. This is *additive* to the judgment routing above.

**Eligibility — cheap-start ONLY read-only or trivially re-runnable subtasks.** The escalation retry re-runs the *same work* at a higher tier, so it is only safe when a failed cheap attempt left nothing half-done:
- ✅ **Eligible:** search / explore, fact-gathering, a self-contained analysis, a first-pass draft you will review, a read-only audit.
- ❌ **NOT eligible — start these heavy:** anything that **mutates shared state, writes files other steps depend on, makes external/irreversible calls, or is destructive.** A partial-write failure leaves changed state the retry can't cleanly redo.

**Escalate only on a POSITIVE failure signal** — an error, an empty/malformed/truncated result, an explicit low-confidence or "cannot decide" from the subagent, or a failed verification/test. **Do NOT treat "looks plausible" as success and do NOT treat it as failure** — escalation catches *observable* failure, not confident-but-wrong output. Confident-but-wrong is still the job of `cross-validate`.

**Cost is not free.** When the cheap attempt fails, its cost is *added on top of* the frontier retry — escalation only nets out when the cheap tier succeeds often enough to outweigh the wasted retries. Whether that holds for a given task class is an empirical question, which is why the loop below measures it.

## The measure → advise → adapt loop (this is where routing self-modifies)

The escalation defaults above start as judgment. Over time, apex-router turns them into **evidence** — and the routing adapts to what's measured, without any artifact rewriting itself.

**1. Log each cheap-start outcome (measure).** After an eligible cheap-start resolves, record whether it succeeded or escalated. Fail-safe (logging never blocks or breaks a dispatch); partial coverage is fine.

```bash
# cheap attempt succeeded:
apex-router route-log --task-type explore  --start-tier sonnet --outcome ok
# cheap attempt failed → you re-dispatched heavy:
apex-router route-log --task-type generate --start-tier sonnet --outcome escalated --note "empty result"
```

`--task-type` is your intent label (explore/generate/review/refactor/debug); `--start-tier` is the cheap model you tried; `--outcome` is `ok` or `escalated`.

**2. Read the escalation rate (readout).** `apex-router route-readout` aggregates the log into a per-task-type escalation rate ("when we start `explore` cheap, how often does it bounce heavy?").

**3. Act only on a SIGNIFICANT rate (advise → adapt).** A raw rate is not actionable — 3/5 escalations is noise. `apex-router route-advise` applies a binomial significance test (Wilson score interval + a minimum-sample floor) and recommends a change **only when the evidence clears a threshold**:

```
apex-router route-advise
# task_type      n   rate      95% CI   recommendation
# explore       60   0.03  [0.01,0.11]  KEEP_CHEAP   ↳ cheap-start reliable
# generate      50   0.90  [0.79,0.96]  START_HEAVY  ↳ cheap-start not paying off
# debug         12   0.58  [0.32,0.81]  HOLD (default) ↳ CI straddles the band / n below floor
```

- **START_HEAVY** — escalation rate is significantly high (cheap-start is wasting a retry most of the time). **For this task-type, stop cheap-starting and route heavy from the start.** The measured cost of the escalation retries outweighs the cheap wins.
- **KEEP_CHEAP** — escalation rate is significantly low. **Keep (or widen) cheap-start for this task-type** — the evidence says it reliably works.
- **HOLD (default)** — not enough samples, or the interval straddles the threshold band. **Keep the static judgment default from the tiers table above.** This is the common, safe case: the loop only overrides your default when the data is strong enough to earn it.

**Beyond the escalation on-ramp: measured chain planning.** `route-advise` decides
*start-cheap-or-not* for a single subtask. For a multi-model **chain** (retrieve→validate→
synthesize), apex-router extends the same measured discipline with a promotion-gate-backed
planner — it proposes which tiers earn their place per task-class and shows the evidence:

```bash
apex-router chain-planner --task-class <cls>   # proposed chain + "Basis: N chains, +Δ (CI…); X below gate"
apex-router chain-bench   --rows <sc2.jsonl>   # per-slot verdict: ON (keep) | OFFERED | SKIP (drop)
```

Same rule as below: it **recommends** a chain with its metrics; you confirm/edit before it
runs. A slot is only dropped (SKIP) on a significant, out-of-sample, FDR-corrected null —
never a raw average.

**This is the self-modification, and its guardrails matter.** The routing *decision* adapts to measured data — but through a reviewable recommendation you read and apply, never a silent auto-flip. Three things keep it honest, and you should respect them:
- **It recommends; it doesn't mutate.** Nothing rewrites this skill or a config. You (or a human) read `route-advise` and change how you route. That keeps a confounded signal from silently degrading routing.
- **Escalation frequency is a proxy, not a full cost model.** It counts how *often* a cheap start fails, not the token cost of each retry. A significant START_HEAVY is a strong signal the cheap start is uneconomic for that task-type — treat it as decisive for the escalation on-ramp, but a full quality+cost model choice (which model is *best*, not just start-cheap-or-not) still belongs to a gated benchmark, not to this rate alone.
- **HOLD is the honest default, not a failure.** Most task-types will sit in HOLD until enough outcomes accumulate. Don't force a flip on a wide interval — the dead band exists on purpose.

## Cross-Validation Is Mandatory (not optional)

Whenever heavy coding or a decision-driving report/analysis is about to be trusted, committed, or delivered, **run the `cross-validate` skill** — an independent, fresh Opus-tier reviewer adversarially refutes what the heavy tier produced. A model reviewing its own work repeats its own blind spots; independent disagreement is the signal you're paying for.

This is the closing step of the heavy path: **heavy tier produces → independent reviewer refutes → reconcile against ground truth before trusting.** Don't skip it for non-trivial changes or customer-facing / go-no-go output just because the heavy tier wrote it.

**Related:** `verify-claims` checks a report's *numbers* against source; `cross-validate` checks the *code/reasoning*. A customer-facing report earns both.

## Worked Examples

**Example 1 — "Find why the metrics cron is failing and fix it."**
- Search (Light, low effort, parallel): grep the cron definition, the wrapper script, recent logs. → conclusions back to you.
- Reason (you, Heavy, high): root-cause from the findings.
- Fix (Heavy, medium–high): the wrapper hardening.
- Cross-validate (mandatory): `cross-validate` on the diff before calling it done.

**Example 2 — "Rename `foo_bar` to `baz` across the pipeline."**
- Pure mechanical → Light, low effort, handles the whole thing; you spot-check. No cross-validation needed (trivial, verifiable).

## Red Flags — STOP

- **You're running the heavy tier at max effort to grep or curl.** Drop to the light tier, low effort; keep your context for reasoning.
- **You're about to let a light model make the call** — the synthesis, the root cause, the go/no-go. Delegate that piece to a heavy subagent; don't let the light tier decide.
- **You're about to switch your session's model to route a single subtask.** STOP — that evicts your prompt cache. Spawn a subagent for the cheap piece instead; keep your session on one model.
- **You're alternating your session model request-to-request.** Worst case for cache. Pick one session model, vary *effort* instead.
- **You raised the model tier but left effort at `low`** (or cranked effort on a model too small to use it) — the dials are mismatched. Set them together.
- **You're about to flip a route based on a raw escalation rate.** Run `apex-router route-advise` first — act on a *significant* recommendation, not a noisy point estimate. HOLD means keep the default.
- **Heavy coding or a decision-driving report is about to be trusted/committed/delivered with only one model's eyes on it.** Run `cross-validate` first — this is not optional.
