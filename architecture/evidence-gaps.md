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
| AA5 | Premises `unresolved-reference` and `call-sites-truncated` | [ADR-A009](decisions/ADR-A009-premise-catalogue.md) | Derived from P10, not from a finding. Correct in spirit; no experiment earned them |
| AA6 | PHP 8.2 and zero runtime dependencies | [ADR-A001](decisions/ADR-A001-php-cli-zero-dependencies.md) | Strongly consistent with the evidence (PSR-4 map input; four PHP commits) but the research never names a language |
| AA7 | The exit-code taxonomy `0 / 1 / 2` | [03-interfaces](03-interfaces.md) | Ordinary CLI practice; no research basis, none needed |
| AA8 | JSON as the default format, and `bundle_version` as a versioned contract | [03-interfaces](03-interfaces.md) | The implementation spec says the serialisation is *not* fixed by it |

None of these may be treated as validated. Each is stated here rather than argued inside the module
that relies on it.
