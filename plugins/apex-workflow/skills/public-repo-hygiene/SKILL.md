---
name: public-repo-hygiene
description: Use before pushing, committing to, or opening a PR against a PUBLIC or shared repository — the last gate before code leaves your machine. Scans the ADDED lines for secrets, internal identifiers, and personal paths, and checks the committer identity, so a private detail doesn't land in public history where deleting it later doesn't help. Symptoms: "about to push to the public repo", "is this safe to make public", "check for leaks before I push", moving private code to an open-source repo, any push to an org/shared remote.
---

# Public-Repo Hygiene — the last gate before code leaves your machine

## Overview

Pushing to a public repository is **irreversible**: once a secret, an internal name, or a customer identifier is in public history, deleting it in a later commit does not remove it — it stays in the history, in clones, in forks, and in mirrors/caches. The only safe time to catch a leak is **before the push**.

**Core principle: scan the ADDED lines of what you're about to push, not the whole tree.** A repo-wide grep drowns you in false positives from code you're not touching (and, on a migration, in *removed* lines that read as leaks but are being deleted). What can leak is what this push *adds*. Diff against the remote, look at the `+` lines, and clear those.

**Leaks hide in the places that aren't "source code."** The dangerous ones this session and past migrations turned up were never in an obvious `API_KEY =` line — they were in test-DATA fixtures, config defaults, provenance comments, auth-flow examples, hardcoded home paths, and internal project/ticket names embedded in docs or specs. A scan that only greps `.py`/`.ts` and skips tests, docs, and fixtures misses exactly where leaks live.

**Two independent things must both pass:** *what* you're pushing (no secrets/internal identifiers in the added lines) **and** *who* is pushing it (the committer identity, not just the author, is your public one — a rebase or a shared machine can silently stamp a corporate email onto a commit).

## When to Use

- Before `git push` / `gh pr create` to any **public** or **shared/org** remote.
- When moving private code into an open-source repo (the highest-risk case — the whole point is that internal work becomes public).
- Before making a private repo public.
- Any time you can't answer "what internal detail could be in this diff?" with certainty.

**When NOT to use:** a purely private repo only you can see, or a push that adds nothing (a merge/ff with no new content). Even then, the identity check is cheap insurance.

## The gate — run all of it, clear every hit

### 1. Scan the added lines for content leaks

Diff against the remote you're pushing to (not `HEAD~1` — you may be pushing several commits), take only added lines, and scan:

```bash
# added lines about to be pushed to origin/<branch>
git diff origin/main..HEAD | grep '^+' | grep -vE '^\+\+\+' | grep -inE \
  'secret|password|api[_-]?key|bearer|token[[:space:]]*[:=]|BEGIN [A-Z ]*PRIVATE KEY|\
[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}|\
/Users/[a-z]|/home/[a-z]|\
[0-9]{1,3}(\.[0-9]{1,3}){3}'
```

Then a **second, project-specific pass** for the internal identifiers a generic regex can't know — the names, ticket prefixes, hostnames, and internal repo names particular to your org. Build this list once and keep it; the generic pass will never catch an internal repo name, a `TICKET-12345` prefix, or an internal cluster/hostname:

```bash
git diff origin/main..HEAD | grep '^+' | grep -vE '^\+\+\+' | grep -inE \
  '<internal-repo-names>|<ticket-prefix>-[0-9]|<corp-domain>|<internal-hostnames>'
```

**Triage every hit — do not wave any through:**
- A real secret/email/internal name in an added line → **stop, remove it, rewrite the commit** (not a follow-up commit — the value must never enter history). If it was ever committed and pushed, treat the secret as compromised and rotate it.
- A false positive (the regex matched `read_tokens=0`, or a variable literally named `token`) → confirm by eye that it's not a value, and move on.
- **The most common real catch is an over-broad `git add`** that staged a local-only file (a design spec, a scratch note, a `.env`) full of internal specifics. Unstage it; it was never meant to ship.

### 2. Verify the committer identity across the whole range

The **committer** email — not just the author — is what appears in public history, and the two can differ. A rebase re-stamps the committer; a shared or work machine may have a corporate `user.email` configured. Check every commit you're about to push:

```bash
git log origin/main..HEAD --format='%h  author:%ae  committer:%ce' \
  | grep -v '<your-public-email>' && echo "⚠ NON-PUBLIC IDENTITY — fix before push" \
  || echo "✓ all commits are your public identity"
```

If any commit is wrong: set the repo identity (`git config user.email <public>`), then `git rebase --reset-author` (or amend) to restamp. Verify `%ce`, not just `%ae` — checking only the author misses the committer trap.

### 3. Confirm the file set is what you intend

List the files in the push range and read the list. A leak often announces itself as a file that shouldn't be there at all:

```bash
git diff origin/main..HEAD --stat
```

Source + tests + docs you meant to add: good. A local spec, a data file, a notebook, a `.env`, a credentials JSON: stop and remove it.

### 4. (High-stakes) Verify from a fresh clone

For a first open-sourcing or a security-sensitive push, the authoritative check is what a stranger who clones the repo actually gets. Push to a branch (or a private staging remote) first, clone it to a clean directory, and grep the clone — no local uncommitted state, no gitignored files masking the truth. What isn't in the fresh clone can't leak.

## Discipline

1. **Scan added lines, against the remote — never a whole-tree grep or a `HEAD~1` diff.** The push range is the blast radius.
2. **Keep a project-specific identifier list.** Generic regexes catch secrets and emails; only your list catches your internal repo names, ticket prefixes, and hostnames. This is the pass that catches the leaks that matter.
3. **Include tests, fixtures, docs, and config in scope.** Leaks hide outside source. A scan scoped to `*.py` is a scan that misses.
4. **Check the committer (`%ce`), not just the author.** The identity trap is a committer-email trap.
5. **A leak in an added line is a rewrite, not a follow-up commit.** Removing it next commit leaves it in history. Amend/rebase so the value never enters history at all; rotate anything already pushed.
6. **The tool's job is to make code provider-neutral, not to smuggle real data into fixtures.** Test fixtures for public code use obviously-synthetic names (`alpha`, `foo_bar`), never real internal filenames — a "realistic" fixture is a leak with a test around it.

## Red Flags — STOP

- About to push to a public/shared remote and you have **not** scanned the added lines this push.
- You ran a broad `git add -A` / `git add .` right before pushing (did it stage a local-only file?).
- You just rebased, cherry-picked, or committed on a machine with a work email configured (check `%ce`).
- A scan hit an internal name/secret and you're about to "fix it in the next commit" — that doesn't remove it from history.
- You're relying on `.gitignore` to keep something out — a fresh clone is the only proof; an already-tracked file ignores the ignore.

**All of these mean: diff against the remote, scan the added lines (generic + your internal-identifier list), verify the committer identity, read the file list — and clear every hit before the push, because after the push there is no clearing it.**
