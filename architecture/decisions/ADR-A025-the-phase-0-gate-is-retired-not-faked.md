# ADR-A025 · The Phase 0 acceptance gate is retired, not faked

- **Status:** Accepted
- **Date:** 2026-09-20
- **Phase:** Phase 1 · closes M28's first item
- **Implements:** ADR-001's key-first method, P6, P8
- **Changes:** the test suite and its documentation only. **No production file, no schema, no
  assertion kind, no premise, no lever, no port, no `bundle_version`.**

## Why this decision exists

`06-acceptance.md` stated Phase 1's acceptance criterion: *for all four Phase 0 commits, the tool's
bundle reproduces the minimum-context list in that commit's experiment file.* M10 built the harness
— `ExperimentKeyTest` — and transcribed the four lists into `expected-context.md` files.

**The harness could never run, and it never did.** Three facts, each checked rather than recalled:

1. The four commits live in a private Laravel + Plaid codebase. No checkout of it exists on any
   machine this project has run on, and the research documents that describe the commits
   (`docs/01-phase0/experiment-0{1..4}.md`) contain **no patch text at all** — zero `diff --git`
   lines, zero hunk headers.
2. `git log --all` over the four fixture directories shows exactly one commit ever touching them,
   `212ea91` (M10), and it added only the four `expected-context.md` files. No `diff.patch` and no
   repository were ever present. Nothing was lost; nothing was dropped.
3. The harness refused to fabricate what it lacked — correctly: *"Fabricating a diff would grade
   the tool against invented ground truth, which is the one thing the answer-key method exists to
   prevent."* So it failed four tests, loudly, on every run from M10 through M27. Seventeen
   milestones of write-ups carry the same sentence: *"4 failures — the four `ExperimentKeyTest`
   experiments, private fixtures absent."*

A gate that cannot open is not a gate. Four permanently red tests train every reader to look past
red, which is the exact condition under which a fifth, real failure goes unnoticed.

## Decision

> **The Phase 0 acceptance harness is retired. Its behavioural content is asserted by tests that
> actually run; its answer keys are preserved as documentation; the process-contract test it also
> carried is kept. Nothing is faked to make it pass.**

### What was never satisfiable inside this repository

The acceptance criterion binds the tool to four inputs this repository does not have and cannot
honestly reconstruct. A synthetic fixture *shaped like* experiment 01 would grade a commit written
here against a key transcribed from a commit never seen here — invented ground truth wearing a real
key's name. That is worse than the red test it replaces.

### What covers the behaviour the keys describe

Checked condition by condition, excluding the retired harness:

| Phase 0 claim | Where it is asserted now |
|---|---|
| `surrounding-transaction` fires on several persistence writes with no local transaction | `UnverifiablePremiseAssertionExtractorTest`, `AssumptionWriterTest`, `BundleAssemblerTest` |
| `atomic-lock-store`, `schema-index-support`, `data-state-after-behaviour-change` | `UnverifiablePremiseAssertionExtractorTest`, `AssumptionWriterTest`, `LeverPolicyTest` |
| a caller search completing with zero call sites is a diagnostic, not an item | `DiscoverContextTest` |
| a changed signature (`reactivate`) resolves to call sites with line-only provenance | `DiscoverContextTest`, `CallerResolverTest`, and five others |
| named references (`createLinkToken`, `REVOKED`) resolve to one member or one surface | `NamedReferenceResolverTest`, `NamedReferenceAssertionExtractorTest` |
| **experiment 02**: a self-contained trace-logging diff must not pull the model "just in case" | **had no named home.** `SelfContainedDiffTest` now asserts it on a synthetic diff that is explicitly not labelled as experiment 02 |
| exit codes `0` / `1` / `2` as a process | was inside the harness; now `ProcessContractTest` — the only exit-code test, and the part of the harness worth keeping |

### What is preserved

The four `expected-context.md` files move to `docs/research/phase0-keys/` with a README stating
plainly what they are: hand-transcribed keys for commits this repository never had. They are the
record of what Phase 0 asked for. Nothing reads them.

### What is retired

`ExperimentKeyTest` in full — the data-provider test, the per-experiment check methods, the scoring
and reporting helpers, and the fixture-completeness gate. The four fixture directories are removed.

## Consequences

**Positive**

- The suite is green with no permanently excused failure. Red means something again.
- One claim that had been asserted only by a test that could not run — experiment 02's precision
  claim — is now asserted by one that does.
- The exit-code contract keeps its only test.

**Negative, stated plainly**

- The literal Phase 0 criterion — *reproduce the four real commits' minimum-context lists* — is no
  longer a test in this repository, because it never could be. If the private codebase ever becomes
  available, the preserved keys are the starting point for a fixture in the `experiment-08…16`
  style, with a real `diff.patch` and a checked-in post-image `repo/`.
- Seventeen milestone write-ups refer to "the four `ExperimentKeyTest` failures." They are dated
  records and are left as written.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Build synthetic fixtures for 01–04 and key them from the Phase 0 lists** | Invented ground truth under a real key's name — the failure the harness existed to prevent |
| **Mark the four tests skipped** | A skip is a quiet excuse; the original design chose loud failure over it for a reason, and a permanent excuse is no better than a permanent failure |
| **Keep the harness and the four red tests** | The user's stated goal was a green suite with nothing permanently excused; and a standing red trains everyone to ignore red |
| **Delete the keys with the harness** | They are the only transcription of what Phase 0 asked for, and they cost nothing to keep as documentation |

## Related

- `06-acceptance.md` §1 — rewritten to describe what the acceptance strategy is now.
- [ADR-A024](ADR-A024-bundle-contract-v2.md) — the golden fixture that replaces "the four keys" as
  the recorded end-to-end artefact, now with its input diff committed beside it.
- `docs/research/phase0-keys/README.md` in the CLI repository.
