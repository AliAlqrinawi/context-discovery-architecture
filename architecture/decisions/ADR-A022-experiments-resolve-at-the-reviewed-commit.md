# ADR-A022 · An experiment resolves against the tree of the commit it is reviewing

- **Status:** Accepted · **Partially superseded by [ADR-A026](ADR-A026-the-harness-places-a-real-vendor-directory.md)** (2026-09-22) — the decision stands; its measured effect and its `vendor/` caveat do not. See the correction note at the end
- **Date:** 2026-09-06
- **Phase:** Phase 1 · corrects a **measurement** defect found by M19 and fixed in M20
- **Implements:** ADR-001's key-first method, P8
- **Changes:** the experiment harness only. **No production file, no schema, no assertion kind, no
  premise, no lever, no port, no `bundle_version`.**

## Why this is an ADR at all

Every ADR before this one records a decision about the tool. This one records a decision about **how
the tool is measured** — and it earns its place because three milestones' numbers were produced by
the defect it corrects. A programme whose whole method is *write the key first, then measure* has to
treat a defect in the measuring instrument as seriously as a defect in the instrument's subject.

## The defect

M18 and M19 generated every bundle with `--repo` pointing at the corpus repository's **working tree
at HEAD**, while `--diff` was a **historical** commit's diff:

```
--diff  <the commit's own diff>        # the past
--repo  <the repository as it is now>  # the present
```

Source resolution therefore used whatever those files look like *today*. For a commit whose files
have since moved, the tool read the wrong file — or, where the path no longer exists at all, no file:

- **6 of M18's 46 commits** carry an `unreadable path` diagnostic for this reason;
- **M19's task K1 was compromised** by it and had to be excluded from that milestone's totals.

The tool did exactly what it was told. It was told the wrong thing.

## Decision

> **An experiment that reviews commit `C` resolves source against `C`'s own tree**, obtained as a
> detached `git worktree`, never against the repository's current HEAD.

The harness is `tests/Acceptance/fixtures/experiment-20/harness/bundle-at-commit.sh`. It adds a
worktree, points `--repo` at it, and removes it on exit — leaving the corpus repository byte-clean.

## What it changes, measured

On M19's compromised task the correction is not cosmetic:

| `0a6e7d1` | items | fetched | flagged | tokens | diagnostics |
|---|---:|---:|---:|---:|---|
| resolved at HEAD (M19) | 2 | 2 | 0 | 221 | 2 × `unreadable path` |
| resolved at the commit's tree | **4** | **3** | **1** | **449** | 1 × `unreadable path`, and a resolved reference |

The one remaining `unreadable path` is **correct**: that commit deletes the file, so it genuinely
does not exist in its own tree.

Across M20's seven candidates the corrected harness produces materially larger bundles than M18/M19
recorded — `ec92403` moves from 18 items / 606 tokens to **37 items / 986 tokens**. **M18's and
M19's bundle sizes are therefore understated**, and their conclusions are correspondingly
conservative rather than inflated. Neither is reopened.

## The residual caveat, stated rather than hidden

`vendor/` is not committed to the corpus repository, so the historical worktree borrows the
installed dependencies from the main checkout by symlink. Composer's generated map is location
metadata that the tool parses and never executes (ADR-A014), so this is sound for placement — but a
commit whose dependency set differed from today's would resolve framework classes against today's
vendor directory. Correcting that needs a `composer install` per commit, which the corpus cannot
support offline. Recorded as a known limit of every measurement taken this way.

## Consequences

**Positive**

- Bundles measured for a historical commit are the bundles that commit would actually have received.
- The failure mode is now pinned by `tests/Acceptance/HarnessResolvesAtCommitTreeTest.php`, which
  builds a throwaway two-commit repository where a class moves, asserts that the HEAD-pointing
  harness cannot see the reviewed file, that the corrected one can, and that the corpus is left
  clean with no worktree behind.

**Negative**

- Every experiment is slower by a worktree add and remove per commit.
- M18's coverage figure and M19's bundle sizes were taken with the defect present. They are
  understated, not wrong in direction, and are left standing with this ADR cited beside them.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| `git stash` / `git checkout` the corpus repository in place | Mutates a repository the experiment does not own, and a crash leaves it on a detached HEAD |
| Reconstruct the tree by applying diffs to a scratch directory | Reimplements `git` badly, and the reconstruction itself becomes something to verify |
| Accept the drift and note it | Tried, in M19. It cost one task outright and silently shrank six others |
| Change the tool to accept a git revision | A production change to fix a harness defect — the opposite of what the evidence supports |

## Related

- `docs/research/M20-defect-corpus.md` — the milestone that found and fixed it.
- `docs/research/M19-scored-review-rerun.md` §6 — where K1 was classified compromised.
- [ADR-A014](ADR-A014-composer-generated-map-as-location-metadata.md) — why borrowing `vendor/` is
  sound for placement.

---

## Correction · 2026-09-22 · see ADR-A026

The body above is unaltered. [ADR-A026](ADR-A026-the-harness-places-a-real-vendor-directory.md)
records, with the measurement in M29, that the harness this ADR introduced **never gave the tool a
`vendor/` it could read**: the symlink under "The residual caveat" was refused whole by
`LocalSourceRepository`, and every bundle from M20 through M23 was generated with no dependency map.

| Passage above | Status |
|---|---|
| **Decision** — resolve against the reviewed commit's own tree | **Stands.** Proof commit `0a6e7d1` is 4 · 449 with and without a vendor |
| "What it changes, measured" — *"`ec92403` moves from 18 items / 606 tokens to 37 items / 986 tokens … M18's and M19's bundle sizes are therefore understated"* | **Superseded.** With a readable `vendor/` at the commit's tree, `ec92403` is **18 · 606**. The 19 extra items were spurious flags on dependency classes |
| "The residual caveat" — *"the historical worktree borrows the installed dependencies from the main checkout by symlink"* | **Superseded.** Nothing was borrowed. The caveat's *content* — today's `vendor/` against a historical commit's dependency set — is **real from ADR-A026 on**, and is restated there as the standing limit |
| Consequences — *"understated, not wrong in direction"* | **Superseded** for M18/M19 bundle sizes. M19's coverage remark about M18 rests on tree drift and stands |
| Consequences — *"pinned by `HarnessResolvesAtCommitTreeTest`"* | **Overstated.** That test never runs the script. `HarnessPlacesAReadableVendorTest` does |
| Related — *"ADR-A014 — why borrowing `vendor/` is sound for placement"* | The premise is false; ADR-A014 itself is correct, and its symlink-refusal sentence is the mechanism |
