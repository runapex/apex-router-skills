# Invoking apex-workflow skills — Pi and Claude CLI

These skills are plain [Agent-Skills-spec](https://agentskills.io) `SKILL.md` files, so
the **same skill runs in both Claude Code and Pi**. Below: how to install and how to
"kick" each one in each tool, plus the apex-router capabilities they now pair with.

## Install

### Claude Code (CLI)
```bash
claude plugin marketplace add runapex/apex-router-skills
claude plugin install apex-workflow@apex-router-skills
```
(apex-router's `./install.sh` does this by default; `--skills-only` re-wires it.)

Skills then load two ways:
- **Automatically** — Claude reads a skill's `description` and pulls it in when your task matches.
- **Explicitly** — force it: `/apex-workflow:model-routing` (plugin-namespaced skill command).

### Pi
Pi discovers skills from `~/.agents/skills/`, `~/.pi/agent/skills/`, and project
`.agents/skills/` / `.pi/skills/`, and can be pointed at the Claude install too. Pick one:

```bash
# A) share the installed plugin's skills with pi (one symlink, stays in sync):
ln -s ~/.claude/plugins/cache/apex-router-skills/apex-workflow/*/skills ~/.agents/skills/apex-workflow

# B) or point pi at the Claude skills dir via settings (~/.pi/agent/settings.json):
#    { "skills": ["~/.claude/plugins/cache/apex-router-skills/apex-workflow/0.3.0/skills"] }
```

Skills then load two ways:
- **Automatically** — the agent loads a skill when the task matches its `description`.
- **Explicitly** — `/skill:model-routing`, `/skill:cross-validate` (models don't always
  auto-load; forcing with `/skill:name` is the reliable way).

## When & how to kick each skill

| Skill | Kick it when… | Claude CLI | Pi |
|-------|---------------|------------|-----|
| **model-routing** | before delegating subtasks / picking a model tier + effort / routing a chain | `/apex-workflow:model-routing` | `/skill:model-routing` |
| **cross-validate** | a non-trivial change/report is about to be trusted/committed/delivered | `/apex-workflow:cross-validate` | `/skill:cross-validate` (or `>>deep`/`>>kimi` for the reviewer) |
| **verify-claims** | any model states a specific number/count/id from data | `/apex-workflow:verify-claims` | `/skill:verify-claims` |
| **disciplined-execution** | a hard, multi-step/unknown/ debugging task | `/apex-workflow:disciplined-execution` | `/skill:disciplined-execution` |
| **public-repo-hygiene** | before pushing/committing to a public/shared repo | `/apex-workflow:public-repo-hygiene` | `/skill:public-repo-hygiene` |
| **local-references** | ground an answer in your own books/code samples | `/apex-workflow:local-references` (or `/books`) | `/skill:local-references` (or `/books`) |

## The apex-router capabilities these skills now pair with

These are the concrete tools the skills route you toward. **Pi** ships first-class
commands; **Claude Code** uses subagents + the `apex-router` CLI + slash commands.

### Per-task model routing (Pi-native, cache-safe)
`model-routing` says "never flip your *session* model per subtask" — because that evicts
your prompt cache. **Pi resolves this** with the `apex-route` extension: a `>>family` cue
runs *one task* on another family and **restores your model automatically** (each family
keeps its own cache, so there's no re-write penalty):
```
>>local  grep for the config loader          # local Ornith tier (ollama)
>>kimi   summarise this diff                  # Kimi K2 (via the apex proxy)
>>frontier design the migration              # Claude Sonnet
>>deep   audit this for race hazards          # Claude Opus
/apex-route            # list families + show the active model (sticky switch)
```
In Claude Code the equivalent is spawning a subagent at the chosen tier (the session
model stays fixed) — the same principle, different mechanism.

### One-command find→verify→synthesize chain (`/learn`)
The canonical routing shape (cheap retrieve → mid validate → heavy synthesize) is a single
Pi command that restores your model at the end:
```
/learn <topic>     # retrieve (local) → sonnet-4-6 validate → opus-4-8 explain, correlated to your code
```

### Measured chain planning (extends `route-advise`)
`model-routing`'s measure→advise→adapt loop now has a promotion-gate-backed planner that
proposes a model chain per task-class and shows the metrics behind it:
```bash
apex-router chain-planner --task-class <cls>   # proposed chain + "Basis: N chains, +Δ (CI…)" rationale
apex-router chain-bench   --rows <sc2.jsonl>   # per-slot gate verdict: ON | OFFERED | SKIP
apex-router route-advise                        # the escalation on-ramp (KEEP_CHEAP/START_HEAVY/HOLD)
```

### Local references (`local-references` / booksearch)
Ground an answer in your own library instead of the model's memory:
```bash
booksearch query "how do B-trees stay balanced?" -k 5   # Top-K books + code samples, local
```
Pi: `/books <topic>` · Claude Code: `/books <topic>` (slash command) — both call the same
local index (nomic-embed retrieval + local-model reasoning).
