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
