# Evidence Gaps

What the source-of-truth repository does **not** settle, and how this architecture responds. No gap
is filled by invention.

## 1 · Documents referenced but missing from the repository

Read at `main@785e3eb811fc`:

| Referenced as | Referenced by | Actual state |
|---|---|---|
| `docs/03-phase0/summary.md` (the evidence table; "what was proven / what stays open") | README, ROADMAP, all four introduction docs, glossary, requirements, all three ADRs | **Absent.** No `summary.md` exists anywhere in the repository |
| `CONTRIBUTING.md` | README | **Absent** |
| `docs/00-project-overview.md`, `docs/01-problem-statement.md`, `docs/02-hypothesis.md` | README reading order, glossary links | Exist as **0-byte** files; the real content is in `docs/00-introduction/` |

Also: the README's repository map uses `03-phase0/`, `04-discovery/`, `05-phase1/`, `decisions/`,
while the tree actually has `01-phase0/`, `02-discovery/`, `03-phase1/`, `04-decisions/`. Every
in-repo cross-link therefore points at a path that does not resolve.

**Response.** This architecture is built from the documents that exist. Nothing was inferred about
the missing `summary.md`; where other documents quote its conclusions (the unscored A/B/C runs, the
recall/precision framing, the Experiment 5 gap), those quotations are used, attributed to the
document that carries them. Fixing the paths and writing `summary.md` is research-repository work,
not architecture work.

## 2 · Numbers the research deliberately leaves unfixed

| Unfixed | Where | Architectural response |
|---|---|---|
| The token budget's absolute value | Hypothesis says only "dramatically below B", "well under half of B's tokens" | `--budget` is **required**, with no default ([ADR-A008](decisions/ADR-A008-required-budget.md)) |
| "Meaningfully more" / "dramatically below" thresholds | Hypothesis, fixed in advance but not numerically stated in the repository | Not encoded anywhere in the tool. Scoring is human, downstream (P6, X4) |
| How large a "minimal slice" may be | Spec says "ideally one method or class member, not a whole file" | Slicer returns exactly the named member, its signature and body; no configurable radius |
| Full-repo token size (B's cost) | Never measured in the repository | Not computed by the tool; comparison is Phase 2's job |

## 3 · Open questions that must stay open

Carried from `requirements.md`, `fetch-vs-flag.md`, and ADR-003, and mechanically kept out of the
design:

1. **Does supplied context improve a review?** Untested. No module weights, ranks, or predicts
   usefulness.
2. **Is the false-positive rate tolerable?** Never scored. Nothing in the tool suppresses or
   trims items to look precise; precision is measured from `assertion_kind` after the fact.
3. **Does a flag capture most of reverse-caller's value?** Never measured. Both levers are recorded
   per item so a scored run can compare them; neither is treated as equivalent.
4. **Is the move-set bounded?** Rests on four polished commits. The extractor set is closed
   ([ADR-A003](decisions/ADR-A003-closed-move-set.md)) so that the sloppy Experiment 5 run *fails
   visibly* rather than being absorbed by a flexible engine.

## 4 · Known under-builds (deliberate, revisit only on new evidence)

| Under-build | Trigger to revisit |
|---|---|
| Config and schema handled by flag, not fetch (X3) | A **second** commit needing config/migration context |
| Reverse-caller is a grep, not a call graph (R3, P9) | A scored run showing grep recall is the limiting factor |
| One-constant token estimate ([ADR-A007](decisions/ADR-A007-token-estimate.md)) | A measured discrepancy that changes a budget decision |
| Unified-diff format only | A real input the parser cannot read |
| Zero callers under `--caller-scope` treated as a settled answer, not a scope-limited premise | An experiment showing a finding whose call sites lie outside the searched scope. Exp 4 treats a search across `app/` as the answer, so no premise is earned today |
| Own-file slices in a **modified** file are not withheld even when the hunk shows them in full ([ADR-A019](decisions/ADR-A019-context-the-diff-already-shows-is-not-fetched.md)) | A key-first experiment. Applying the visibility rule there today would suppress M1's `S10-missing-import-absence`, whose `record()` sits inside its own hunk — and with it Experiment 1's headline finding |
| A genuine **missing import in a created file** is no longer reported ([ADR-A018](decisions/ADR-A018-a-created-file-is-input-not-context.md)) | A key-first experiment containing a created file that uses a symbol it does not import. The justification — that a created file's `use` block sits in the diff directly above the code, unlike a modified file's — is a judgement, not a measurement |
| 94 Eloquent dynamic-dispatch flags in one real application, whose fixed statement is arguably false of them ([ADR-A016](decisions/ADR-A016-oq6-the-available-facts-cannot-classify.md)) | Member-level evidence that does not exist inside one file. `Model` publishes no `@method`/`@mixin` surface, so unlike a facade there is nothing to read. Either ADR-A010's bound changes on new evidence, or the reference is settled by something other than resolution |
| One hop to a **dependency** parent — 21 real sites in one application (`XResource::collection` -> `JsonResource.php:86`), precision-only and zero token cost under ADR-A012/A013 ([ADR-A015](decisions/ADR-A015-depth-boundary-trigger-evaluated-and-not-met.md)) | A **key-first** experiment that contains the case before the key is written. It was found while measuring M8, after that experiment's key was fixed, so acting on it now would be the confirmation bias ADR-001 exists to prevent |
| A class surface fetched for a reference that originates in a **test** file — 3 slices / 165 tokens for `User` on M7's real pull request, wanted by no answer-key row ([ADR-A020](decisions/ADR-A020-the-project-class-surface-fallback.md)) | A **key-first** experiment asking whether the file a reference originates in changes what the reference is worth. It was found while measuring M14, after that milestone's key was fixed; "test files are scaffolding" is a plausible new discriminator with no experiment behind it, which is what ADR-A003 exists to stop |
| Resolution opens at most one file beyond the changed file: inheritance (`extends`, `use <Trait>`) and annotation (`@mixin`) chains are not followed, even to *verify* that a member exists ([ADR-A010](decisions/ADR-A010-inheritance-and-annotation-traversal.md)) | An **ADR-001-grade experiment** whose hand-written answer key names a member the directly-referenced class does not itself declare. A count of false positives is a motive to investigate, not an answer key |

Each of these is an under-build with a named trigger — not a backlog item.

## 5 · Architectural assumptions

Recorded by [REVIEW-freeze-01.md](REVIEW-freeze-01.md). Each is defensible and none expands scope,
but none is traceable to a Phase 0 finding. They are listed as assumptions so no reader mistakes
them for evidence.

| # | Assumption | Where | Why it is not evidence |
|---|---|---|---|
| AA1 | `--max-call-sites` default `20` | [03-interfaces](03-interfaces.md), [ADR-A006](decisions/ADR-A006-grep-not-graph.md) | A bound is required by P7/P10; the number is invented. Exp 4 gives no count |
| AA2 | A Markdown writer exists alongside JSON — and is what justifies the `BundleWriter` interface | Adapters | Supported only indirectly: the ROADMAP calls the output something "a human … pastes alongside the diff". The spec leaves serialisation open. Dropping Markdown would also drop the interface |
| AA3 | `assertion_kind` on every bundle item | [03-interfaces](03-interfaces.md) | R4 requires payload, reason, lever. Machine-readable per-move attribution is an addition; the *reason* field is what the research requires for measurability |
| AA4 | The characters-per-token ratio | [ADR-A007](decisions/ADR-A007-token-estimate.md) | Adequate for a coarse comparative claim; unmeasured |
| AA5 | Premises `unresolved-reference`, `caller-search-failed` and `call-sites-truncated` | [ADR-A009](decisions/ADR-A009-premise-catalogue.md) | Derived from P10, not from a finding. Correct in spirit; no experiment earned them. `caller-search-failed` was added at freeze review 06 under the same standing |
| AA6 | PHP 8.2 and zero runtime dependencies | [ADR-A001](decisions/ADR-A001-php-cli-zero-dependencies.md) | Strongly consistent with the evidence (PSR-4 map input; four PHP commits) but the research never names a language |
| AA7 | The exit-code taxonomy `0 / 1 / 2` | [03-interfaces](03-interfaces.md) | Ordinary CLI practice; no research basis, none needed |
| AA8 | JSON as the default format, and `bundle_version` as a versioned contract | [03-interfaces](03-interfaces.md) | The implementation spec says the serialisation is *not* fixed by it |
| AA9 | The token list that counts as a "persistence-write call" in the `surrounding-transaction` trigger | [ADR-A009](decisions/ADR-A009-premise-catalogue.md) | Exp 1 earned the premise from one reconciliation path; the general list of call shapes is not enumerated by any experiment. Experiment 4 is the precision guard — if the trigger fires there, the list is wrong |
| AA14 | Treating "the member does not resolve in the class's own file" as sufficient reason to fetch that class's **surface**, when the class is the project's own ([ADR-A020](decisions/ADR-A020-the-project-class-surface-fallback.md)) | [ADR-A020](decisions/ADR-A020-the-project-class-surface-fallback.md), M13 | Experiment 1 earned the surface for a model reached by name, not for a member that failed to resolve. M13 measured the three alternatives and each produced a false positive, so this is the narrowest boundary that fires at all — but it is still a bridge from *"the member is unaccounted for"* to *"the class surface is what the region can be read against"*, which no experiment states directly. The correction, if it misfires, is a fourth condition on the reference's origin (see section 4), not a wider rule |
| AA11 | Recording a successful negative result (zero call sites, no `use` block) as one stderr diagnostic | [03-interfaces](03-interfaces.md), freeze review 05 | The research requires no such output. It exists so the acceptance harness can distinguish "searched, found none" from "never searched" — the assertion Experiment 1 already needs. No bundle effect |
| AA13 | Treating a path inside Composer's `config.vendor-dir` (default `vendor`) as **not this project's source**, which is what confines the bare-class surface move to the project's own classes ([ADR-A012](decisions/ADR-A012-surface-move-is-for-project-classes.md)) | [ADR-A012](decisions/ADR-A012-surface-move-is-for-project-classes.md) | Composer's own convention, read from the manifest the tool already loads, and not framework-specific. But the research names no such distinction. The correction, if it misfires, is to read ownership from `autoload` versus the generated dependency map rather than from the path |
| AA12 | Counting resolution depth in **files opened**, so that "one file beyond the changed file" is the bound ([ADR-A010](decisions/ADR-A010-inheritance-and-annotation-traversal.md)) | [ADR-A010](decisions/ADR-A010-inheritance-and-annotation-traversal.md) | P3 says "depth one" without naming a unit. Files is the unit `05-traceability.md` §5 and freeze review 01 already reason in, and the unit `SourceRepository.text(path)` operates on — but the research fixes no unit, and counting *classes* instead would silently permit an ancestry walk. Recorded so no reader mistakes the choice of unit for evidence |
| AA10 | Widening the `data-state-after-behaviour-change` trigger from `SoftDeletes` to any trait removal inside a class body | [ADR-A009](decisions/ADR-A009-premise-catalogue.md) | Exp 3 earned it for `SoftDeletes` specifically (n=1). Narrowing back to that single trait is the correction if a fixture shows a false positive |

None of these may be treated as validated. Each is stated here rather than argued inside the module
that relies on it.
