---
name: dependency-vetting
description: >-
  Use BEFORE adding, replacing, or upgrading any third-party package — runtime, test, or build — and before accepting a lockfile change you did not author. Runs a fixed six-point check on the package AND every new or changed package in its transitive lock: official index, exact pinned + hash-locked version, maintained (a release in the last 24 months), permissive OSI license, no known vulnerability per an OSV query you archive, and scoped to the component that needs it. Symptoms: "just pip/npm/cargo install X", a PR whose lockfile diff is longer than its code diff, a model that suggests a package you have never heard of, a stdlib-only component about to grow a dependency.
---

# Dependency Vetting

## Overview

A dependency is code you run with your privileges, written by someone you have not met,
updated on a schedule you do not control. Adding one is cheap to type and expensive to be
wrong about: a typosquat, an abandoned package, a copyleft license in a proprietary tree, or
a known CVE pulled in three levels down. Models make this worse — they suggest plausible
package names that do not exist or are not the one you mean.

**Core principle: a package earns its place by passing six checks, and so does every package
it drags in.** The transitive lock is the real diff. A clean top-level package with a
vulnerable sub-dependency is a vulnerable change.

## When to Use

- Before `pip install` / `uv add` / `npm install` / `cargo add` / `go get` of anything new.
- Before a version bump that changes the transitive lock.
- Before accepting a lockfile change in a PR you did not write (a bot, a subagent, a model).
- When a stdlib-only or minimal-deps component is about to grow a dependency — that is a
  design decision first, and this check second.

**When NOT to use:** a bump within an already-vetted package where the lock diff touches only
that package's own version (re-run step 5 for the new version; skip the rest).

## The six checks

Run all six for the requested package **and** for every new or changed package in the
resolved lock. A check that cannot run is BLOCKED (see `evidence-labels`), never a pass.

| # | Check | Pass condition | How |
|---|---|---|---|
| 1 | **Official index** | comes from the ecosystem's canonical registry (PyPI, npm, crates.io, Go proxy, Maven Central), not a URL, a Git ref, or a mirror you did not configure on purpose | read the lock: source/registry field per package |
| 2 | **Pinned and hash-locked** | exact version in the lock, with content hashes (`uv.lock`/`poetry.lock` hashes, `package-lock.json` integrity, `Cargo.lock` checksum, `go.sum`) | read the lock; a lock with no hashes is not a lock |
| 3 | **Maintained** | at least one release within the last 24 months, and the repository is not archived | registry metadata (`upload_time`, `time`, `created_at`); repo page |
| 4 | **License** | permissive and OSI-approved: MIT, BSD-2/3, Apache-2.0, ISC, PSF, MPL-2.0 if your policy allows file-level copyleft. Anything else — GPL/LGPL/AGPL, SSPL, BUSL, "custom", missing — goes to the user | registry metadata + the LICENSE file in the sdist/tarball (metadata lies; the file does not) |
| 5 | **No known vulnerability** | OSV returns no advisory for that exact version; the response is archived with the change | query below |
| 6 | **Scoped** | declared only by the component that needs it (the member `pyproject.toml`, the workspace crate, the sub-package), not hoisted to the root "to be safe" | read the manifest diff |

### Step 5: the OSV query

[OSV](https://osv.dev) aggregates advisories across ecosystems. One HTTPS POST per package
version, with curl or the standard library — never by installing a scanner, which is itself
an unvetted install:

```bash
# ecosystem: PyPI | npm | crates.io | Go | Maven | RubyGems | Packagist | NuGet
curl -sS -X POST https://api.osv.dev/v1/query \
  -H 'content-type: application/json' \
  -d '{"package":{"name":"<name>","ecosystem":"<ecosystem>"},"version":"<exact version>"}' \
  | tee osv-<name>-<version>.json
```

- `{}` (empty object) means no known advisory for that version. Anything with `vulns` is a hit:
  read each advisory's `affected[].ranges` to confirm your version is inside, then stop and
  report — a hit is the user's decision, not yours.
- Archive every response next to the change (the PR, the blocker log, `bench-results/`),
  named by package and version, so the evidence outlives the session.
- Batch the transitive set with `/v1/querybatch` when the lock diff is large.
- The query needs network. In an offline build (`UV_OFFLINE=1`, `npm ci --offline`), resolve
  the lock in a scratch copy, run the queries, and only then commit the lock. **OSV
  unreachable is BLOCKED**, not a pass.

### Getting the transitive set

Diff the lock, not the manifest. The lines that matter are the packages that appear or change
version:

```bash
git diff -- uv.lock | grep -E '^\+name = '            # Python (uv)
git diff -- package-lock.json | grep -E '^\+\s+"node_modules/'   # npm
git diff -- Cargo.lock | grep -E '^\+name = '         # Rust
git diff -- go.sum | grep -E '^\+' | cut -d' ' -f1-2 | sort -u    # Go
```

Every name on that list goes through checks 1–5. (Check 6 applies to the direct dependency.)

## Discipline

1. **Verify the package is the one you mean.** Models invent names and typosquats exist.
   Open the registry page; confirm the repository link, the maintainer, and that the README
   describes the thing you want. `requests` vs `request`, `python-dateutil` vs `dateutil`.
2. **Read the lock diff before the code diff.** If the lock diff is longer, the dependency is
   the change.
3. **Metadata is a claim; files are evidence.** A license field says MIT; open the LICENSE in
   the artifact. A "last release" date comes from the registry, not the package's own README.
4. **Record the exception where the dependency lives.** A one-line comment in the manifest
   (`# vetted 2026-10-02: OSV clean, MIT, last release 2026-07`) and, if the project keeps a
   state-of-the-world or decisions file, an entry there. The next reader should not have to
   redo the check to know it was done.
5. **A stdlib-only component stays stdlib-only unless the user says otherwise.** Creating a new
   component just to host a dependency is a scope change, not a workaround.
6. **In an unattended run**, the six checks plus one advisor AGREE (see `unattended-loop`)
   are required; a license outside the permissive list, an OSV hit, or an unreachable OSV is a
   hard stop that waits for the user.

## Report template

```
Dependency: <name>==<version>  (direct, scoped to <component>)
Transitive new/changed: <n> packages — list attached
  1 official index     PASS  all from PyPI
  2 pinned + hashed    PASS  uv.lock sha256 for all <n+1>
  3 maintained         PASS  oldest last-release 2025-03 (<pkg>)
  4 license            PASS  MIT ×4, Apache-2.0 ×2, BSD-3 ×1   | FAIL <pkg> is LGPL-3.0 → user
  5 OSV                PASS  <n+1> queries, all {} — archived under <path>
  6 scoped             PASS  declared in <component>/pyproject.toml only
Decision: add | hold for user (<reason>)
```

## Red Flags

- **"It's a well-known package."** Well-known packages have CVEs too. Run step 5; it takes
  seconds.
- **A lockfile change with no manifest change.** Something resolved differently. Find out what.
- **A license field of "UNKNOWN", "Other", or blank.** Open the artifact. If there is no
  LICENSE file, the package is unlicensed and goes to the user.
- **Installing a scanner to run the scan.** That is an unvetted install. OSV is one curl.
- **OSV timed out, so I skipped it.** BLOCKED. Say so; do not merge on it.
- **A model told you the package name.** Confirm it exists and is the right one before
  anything else.

## Pairs with

- `evidence-labels` — the status vocabulary for each of the six checks.
- `unattended-loop` — the only way a dependency gets added without a human: six checks plus
  one advisor round, with license, OSV hits, and unreachable OSV as hard stops.
- `verify-claims` — the version, date, and license you quote came from the registry, not from
  memory.
