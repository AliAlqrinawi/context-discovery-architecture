# ADR-A026 · The harness places a real `vendor/` directory, because a symlinked one is refused whole

- **Status:** Accepted
- **Date:** 2026-09-22
- **Phase:** Phase 1 · corrects a **measurement** defect present from M20 through M23, found while
  designing Phase 3 and measured by M29
- **Implements:** ADR-001's key-first method, P8; **partially supersedes ADR-A022** (its measured
  effect and its caveat, not its decision)
- **Changes:** the experiment harness and its tests. **No production file, no schema, no
  assertion kind, no premise, no lever, no port, no `bundle_version`.** No recorded artefact is
  rewritten; every affected record carries an appended, dated erratum.

## Why this decision exists

ADR-A022 moved the experiments onto a detached worktree of the reviewed commit. `vendor/` is not
committed to any corpus, so the harness linked the main checkout's into the worktree:

```sh
if [ -d "$REPO/vendor" ] && [ ! -e "$WT/vendor" ]; then ln -s "$REPO/vendor" "$WT/vendor"; fi
```

The tool never read it. `LocalSourceRepository` resolves every path through `realpath()` and
refuses one that lands outside `--repo` — the guard ADR-A014 relies on to make the generated-map
read safe. A link to a directory outside the worktree resolves outside the worktree, so
`vendor/composer/autoload_psr4.php` read as absent, and `ComposerPsr4ClassLocator` fell back to the
project's own map. That fallback is deliberately silent (ADR-A014: *"absent, unreadable or
unparseable metadata contributes nothing and raises nothing"*). The tool behaved as on a bare
checkout, and the only trace was the one a genuinely missing dependency leaves: a `missing PSR-4
entry` diagnostic and a `named_reference / flagged` item per dependency class.

Four milestones — M20, M22, M23 and the M21 corpus tooling that shares the harness — ran through
that line. The harness's own comment said the tree *"borrows the one on disk"*; ADR-A022 recorded a
caveat about *today's* vendor being used for a *historical* commit. Both described a condition that
never obtained.

## The evidence

M29 ran the harness itself — not the reader it wraps — on all seven M20 commits and ADR-A022's proof
commit, three ways: **A** the committed script with its symlink, **B** the same script with the one
line changed to `cp -R`, **C** the vendor line removed. Engine `v0.2.0`, corpus `abouelsid-backend`
at `450d91f` with a real `vendor/` installed. Outputs are committed under
`tests/Acceptance/fixtures/experiment-29/`.

| Commit | A vs C | A | B | only in A | only in B |
|---|---|---:|---:|---:|---:|
| D1 `ec92403` | **byte-identical** | 37 · 986 | **18 · 606** | 19 | 0 |
| D2 `fc573d2` | byte-identical | 2 · 40 | **0 · 0** | 2 | 0 |
| D3 `e5e48ce` | byte-identical | 85 · 4694 | 57 · 4134 | 28 | 0 |
| D4 `ec76dc4` | byte-identical | 66 · 3386 | 65 · 3366 | 1 | 0 |
| D5 `44726d0` | byte-identical | 97 · 2700 | 78 · 2320 | 19 | 0 |
| C1 `4411454` | byte-identical | 16 · 479 | **7 · 299** | 9 | 0 |
| C2 `e770086` | byte-identical | 30 · 504 | 27 · 444 | 3 | 0 |
| proof `0a6e7d1` | — | 4 · 449 | 4 · 449 | 0 | 0 |

- **7 of 7: the symlinked run is byte-identical, bundle and stderr, to the run with no `vendor/`.**
- Arm A reproduces every recorded M20 capture exactly, so the record was made in that condition.
- All 81 items in A and not in B are the same item: `named_reference`, `flagged`, *"could not be
  resolved on disk"*, subject a dependency class. B is a strict subset of A in every commit.

## Decision

> **The experiment harness places `vendor/` in the worktree as a real directory — a copy — never
> a symlink. A test runs the harness script itself and fails if that stops being true.**

`bundle-at-commit.sh` now reads `cp -R "$REPO/vendor" "$WT/vendor"`. `git worktree remove --force`
takes the copy with it, so the corpus is left as clean as before.

### What this does to ADR-A022

**ADR-A022's decision stands.** Resolving at the reviewed commit's own tree is right, and its proof
commit `0a6e7d1` is 4 · 449 with and without a vendor — its improvement over M19's 2 · 221 is the
tree. Two parts of ADR-A022 are superseded:

1. **Its measured effect.** *"Across M20's seven candidates the corrected harness produces
   materially larger bundles than M18/M19 recorded — `ec92403` moves from 18 items / 606 tokens to
   37 items / 986 tokens. M18's and M19's bundle sizes are therefore understated."* With a readable
   `vendor/` at the commit's tree, `ec92403` is **18 · 606** — M18's figure exactly, and the figure
   the same diff gives against the main checkout today. The growth was the vendor disappearing.
   D2 and C1 return to M18's figures the same way. No direction is claimed for D3, D4, D5 and C2,
   which are confounded by engine changes since M18.
2. **Its caveat, which is now real for the first time.** ADR-A022 said the historical worktree
   *"borrows the installed dependencies from the main checkout … a commit whose dependency set
   differed from today's would resolve framework classes against today's vendor directory."* That
   was false when written — nothing was borrowed — and is **true from this ADR on**: the copy is
   today's `vendor/`, placed in the tree of a historical commit. A commit that added, removed or
   upgraded a dependency resolves its framework references against the current install, and the
   generated map (ADR-A014) may place a class in a file that did not exist, or existed elsewhere,
   at that commit. Correcting that needs a `composer install` per reviewed commit, which the
   corpora cannot support offline. It is recorded here as the standing limit of every measurement
   taken through this harness, and it is the caveat ADR-A022 meant to record.

### What "pinned by" pinned

M20, ADR-A022 and this repository's README say the correction is *"pinned by
`tests/Acceptance/HarnessResolvesAtCommitTreeTest.php`."* That test adds its own worktree with
`git worktree add` and **never executes `bundle-at-commit.sh`**. It pinned the idea — resolve at the
commit's tree — and left the script free to be wrong about everything else. Four milestones ran
through the link without a red test. "Pinned by" overstated it.

`tests/Acceptance/HarnessPlacesAReadableVendorTest.php` runs the script against a synthetic
repository whose gitignored `vendor/` holds Composer's generated map and one PSR-4 package, and
asserts that a reference to that package is **settled with its location cited** (ADR-A013) and
that no *"could not be resolved on disk"* flag appears. Reverting the script to `ln -s` fails it at
the first assertion; this was verified by reverting it once. A second test symlinks a `vendor/` by
hand and asserts the flag *does* appear, so the suite records the failure mode as well as the
guard. `HarnessResolvesAtCommitTreeTest` is kept and should be read as a test of the tool on a
moved file, not of the script.

## Consequences

**Positive**

- Bundles generated through the harness now see the dependency map, so a dependency class is
  settled (ADR-A012, ADR-A013) rather than flagged — the behaviour every production run of the
  tool on an installed checkout has always had.
- The harness has a test that runs the harness.
- The corpus repository's location, which no milestone had written down, is recorded in the
  harness header.

**Negative, stated plainly**

- **Every bundle-size and cost figure from M20 through M23 is wrong**, and the sentence *"M18's
  and M19's figures are understated"* is falsified. M29 §9 classifies every conclusion in M19–M23:
  every conclusion counted from reviewer decisions **stands**; every size and cost figure is
  **falsified**; three tasks — M20 D2, M22 K07, M23 C5 — entered their corpora with bundles made
  only of spurious flags and would have failed their own non-empty criterion; two of those flags
  (D2-B, K07-B) were counted as verified bundle-only evidence. Whether the flags changed any
  reviewer's answer — M20 D5 and M22 K11 most of all — is **unknown** and is a Phase 6 question.
- The M22 and M23 corpora were not re-run; their corrected figures are estimates from a namespace
  heuristic exact on six of seven measured commits.
- The tool cannot tell *"no vendor"* from *"vendor refused"*. That is ADR-A014's silent fallback
  and P10's flag-not-silence rule working as designed, and it is what let this go unnoticed. Whether
  a `vendor/` present as a symlink deserves its own diagnostic is a question for the key-first
  gate (ADR-A003), not for this correction.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Teach `LocalSourceRepository` to follow a symlink into `vendor/`** | A production change to fix a harness defect — the same objection ADR-A022 raised to changing the tool. The symlink refusal is also the guard that keeps every read inside `--repo`; an exception for one directory is an exception to root-scoping |
| **A bind mount or hard-linked tree** | Platform-specific, and still not what a real checkout looks like. A copy is exactly what `composer install` leaves |
| **`composer install` per reviewed commit** | The correct answer to the caveat above, and unavailable offline for the corpora. Deferred, not rejected; the Phase 3 backend's `vendor_mode` is where it belongs |
| **Rewrite ADR-A022 and the M20–M23 write-ups** | A recorded milestone is what its reviewers saw and what its author concluded from it. Rewriting would make the correction invisible. Each record keeps its text and gains a dated erratum pointing here |
| **Re-score the affected cells now** | It would answer the open question, but it is Phase 6's comparison — same diff, two conditions — and belongs there with the A and B bundles already committed as its first pair |

## Related

- [ADR-A022](ADR-A022-experiments-resolve-at-the-reviewed-commit.md) — the decision this partially
  supersedes; its status line and appended note point here.
- [ADR-A014](ADR-A014-composer-generated-map-as-location-metadata.md) — the read that was refused,
  and the silent fallback that hid it.
- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md), [ADR-A013](ADR-A013-dependency-members-are-settled-not-fetched.md)
  — what a dependency reference produces when the map is readable.
- `docs/research/M29-vendor-symlink-correction.md` (implementation repository) — the measurement
  and the M19–M23 classification.
- `tests/Acceptance/fixtures/experiment-29/`, `tests/Acceptance/HarnessPlacesAReadableVendorTest.php`,
  `tests/Acceptance/fixtures/experiment-20/harness/bundle-at-commit.sh` (implementation repository).
