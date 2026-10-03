# apex-router-skills

Public workflow-discipline skills that pair with [apex-router](https://github.com/runapex/apex-router). Vendor-neutral discipline for verifying model output and reviewing your own work before you trust it — no external CLI or paid service required. **Runs in both Claude Code and Pi** — see [INVOCATION.md](plugins/apex-workflow/INVOCATION.md) for how to install and kick each skill in either tool.

## Skills

| Skill | What it does |
|---|---|
| **model-routing** | Pick the model tier + reasoning effort per subtask when delegating to subagents or a workflow; route by delegation (not by flipping your session model, which evicts the prompt cache). Includes the measure → advise → adapt loop that lets apex-router's escalation defaults self-tune from logged outcomes. |
| **verify-claims** | Before quoting a model-produced number, count, ID, or filename, check it against the source with a deterministic tool. Grep beats trust. |
| **cross-validate** | Before committing a non-trivial change or shipping a report, have a fresh higher-tier reviewer adversarially refute it — reconciles across Claude, Codex, and Kimi families (produce on one family, review on another). Take the *finding*, author the *fix* yourself. |
| **disciplined-execution** | A five-gate loop (scope → evidence → adversarial reasoning → verify → report) for multi-step tasks, debugging, and review. |
| **public-repo-hygiene** | The last gate before a push to a public/shared repo: scan the *added* lines for secrets, internal names, and personal paths; verify the committer identity; read the file list. After the push there's no taking it back. |
| **local-references** | Ground an answer in your OWN library — books, papers, code samples — via apex-router's `booksearch` (local retrieval + cited passages) instead of the model's memory. Pi: `/books`; Claude: `/books`. |
| **change-classification** | A per-change risk read from a *panel* of independent models: classify a diff on change-class, requirement-fit, and blast-radius from the diff + test output + requirements, and surface where the models AGREE (trust) vs DISAGREE (look here). A measurement, not a gate. Backed by apex-router's `scripts/change_classifier.py`. |
| **evidence-labels** | Nine labels for every check you report — PASS, FAIL, PARTIAL, BLOCKED, INCONCLUSIVE, STALE, ASSUMED, N/A, WAIVED — and one rule: a gap blocks the claim that rests on it, nothing else. Turns "tests pass" into a handoff someone can act on. |
| **unattended-loop** | How an agent decides when nobody is there: a blocker procedure (document → one advisor round → re-check → act → verify), a never-loosen rule for failing gates, a hard-limit list that advisor agreement cannot unlock, scoped commits on `nightly/<date>`, and a morning report. Pairs with `apex-router pressure --check` and `nightly`. |
| **dependency-vetting** | Six checks before any package is added or bumped — official index, pinned + hash-locked, maintained, permissive OSI license, no OSV advisory (one archived curl, no scanner install), scoped to the component that needs it — run on the package *and* every new or changed package in its transitive lock (shift-left) — plus a cadence re-scan of the whole lock diffed against the last archive, because advisories arrive after you vetted (shift-right). |

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add runapex/apex-router-skills
/plugin install apex-workflow@apex-router-skills
```

For **Pi**, point it at these skills (one symlink) and invoke with `/skill:<name>`:
```
ln -s ~/.claude/plugins/cache/apex-router-skills/apex-workflow/*/skills ~/.agents/skills/apex-workflow
```
Full instructions for both tools: [INVOCATION.md](plugins/apex-workflow/INVOCATION.md).

## License

See LICENSE.
