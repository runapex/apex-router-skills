# apex-router-skills

Public Claude Code skills that pair with [apex-router](https://github.com/runapex/apex-router). Vendor-neutral workflow discipline for verifying model output and reviewing your own work before you trust it — no external CLI or paid service required.

## Skills

| Skill | What it does |
|---|---|
| **model-routing** | Pick the model tier + reasoning effort per subtask when delegating to subagents or a workflow; route by delegation (not by flipping your session model, which evicts the prompt cache). Includes the measure → advise → adapt loop that lets apex-router's escalation defaults self-tune from logged outcomes. |
| **verify-claims** | Before quoting a model-produced number, count, ID, or filename, check it against the source with a deterministic tool. Grep beats trust. |
| **cross-validate** | Before committing a non-trivial change or shipping a report, have a fresh Opus-tier Claude reviewer adversarially refute it. Take the *finding*, author the *fix* yourself. |
| **disciplined-execution** | A five-gate loop (scope → evidence → adversarial reasoning → verify → report) for multi-step tasks, debugging, and review. |
| **public-repo-hygiene** | The last gate before a push to a public/shared repo: scan the *added* lines for secrets, internal names, and personal paths; verify the committer identity; read the file list. After the push there's no taking it back. |

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add runapex/apex-router-skills
/plugin install apex-workflow@apex-router-skills
```

## License

See LICENSE.
