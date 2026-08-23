---
name: verify-claims
description: >-
  Use when any LLM — a subagent, a local model, or you — states a specific number, count, filename, ID, or factual claim drawn from data, before quoting it in a report, deciding on it, or telling the user it's true. Symptoms: a fluent model output with concrete figures, a "root cause" with a metric, a claim that "X appears N times" or "the log shows Y".
---

# Verify Claims Against Source

## Overview

**A fluent, confident, plausible claim with a specific number is exactly the kind that is wrong.** LLMs interpolate — they emit numbers that *fit the pattern* whether or not they're in the data. Before a model-produced figure is quoted, acted on, or reported as true, check it against the actual source with a deterministic tool (grep/count/query).

**Core principle: grep beats trust. A number you didn't verify is a number you're guessing.**

## When to Use

- Any model output — a subagent, a local model, or your own — contains counts, percentages, IDs, filenames, timestamps, versions.
- A "root cause" or finding cites a metric ("~59s vs ~11ms", "coverage=0", "134.7% of soft limit").
- Any claim of the form "X occurs N times", "the log/report shows Y", "the dominant pattern is Z".
- Before it goes into a customer-facing report, a go/no-go decision, or a "this is done, it works" statement.

**When NOT to use:** the model is producing prose/opinion/structure with no checkable atoms; or the number was itself computed by an audited deterministic pipeline (then verify the *pipeline*, not re-grep every row).

## The Discipline

1. **Extract the atomic claims** — every number, ID, filename, template, verbatim string.
2. **Pick the source of truth** — the raw log, the JSON the pipeline wrote, the DB, the file. NOT another model.
3. **Check each with a deterministic tool** and report the verdict side by side:
   - counts → `grep -c` / `grep -o … | sort | uniq -c`
   - a quoted line exists → `grep -F`
   - a value in structured data → parse the JSON/CSV and compare the field
4. **Report MATCH / MISMATCH per claim**, values shown. If a claim can't be checked, say so — do not assume it's true.
5. **Distinguish the claim from an artifact.** A number can be *real in the slice/window the model saw* but false globally (e.g. a counter "frozen" only because the sample was small). Verify at the scope the claim asserts.

## Quick Reference

| Claim shape | Verify with |
|---|---|
| "appears N times" / "N% of lines" | `grep -c PATTERN file` ; `grep -oE … \| sort \| uniq -c` |
| "the log shows this exact line" | `grep -F 'the line' file` |
| "value X for field F" (structured) | parse JSON/CSV, print `F`, compare |
| "template T is dominant/rare" | mask variables, `sort \| uniq -c \| sort -rn` |
| "version / target / ID is X" | grep the config/inventory it came from |

## Example (real, this workflow)

A local model summarized `nginx_error.log`: *"Nginx is crashing due to too many open files."* Fluent and plausible. Verify:

```bash
grep -ic "too many open files" nginx_error.log     # -> 0   ❌ FALSE
grep -oE "\[(alert|error)\]" nginx_error.log | sort | uniq -c
#  -> 1599 [alert], 8 [error]   (the log is 'exited on signal 1', not open-files)
```

The claim was a hallucination; the grep took five seconds and caught it. Every specific figure another model quoted (VEN counts, saturation %, "212987") was checked the same way — some MATCHED exactly, one was a slice-locality artifact. **Report the verdict, not the model's fluency.**

## Rationalization Table

| Excuse | Reality |
|--------|---------|
| "The number looks right / plausible" | Plausible is how a wrong interpolation disguises itself. Check it. |
| "It's a strong model, it read the data" | Strong models hallucinate specific figures most confidently. Grep. |
| "I don't have time to verify each one" | One `grep -c` is seconds. A wrong number in a go/no-go costs hours. |
| "It matched the last three, this one's fine" | Verification is per-claim. The fourth is the one that's wrong. |
| "The model even quoted the line" | Quoted ≠ present. `grep -F` the quote. |
| "It's just for internal notes" | Wrong numbers propagate. Internal today, customer-facing next week. |

## Red Flags — STOP and grep

- About to paste a model's number into a report, Confluence doc, or message to the user.
- Writing "the root cause is … (metric)" where the metric came from a model, not a query.
- Saying "done / verified / it works" without having run the check yourself this session.
- Treating a per-slice / per-window observation as a whole-dataset fact.

**All of these mean: check the source with a deterministic tool before the claim leaves your hands.**
