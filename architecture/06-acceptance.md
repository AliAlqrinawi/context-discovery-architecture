# 06 · Acceptance and Test Strategy

The acceptance test is not invented here. It is stated in
`docs/03-phase1/implementation-spec.md`: Phase 1 is done when, for all four Phase 0 commits, the
tool's bundle reproduces the **minimum-context list** in that commit's experiment file — fetching
the fetch-type items and flagging the flag-type ones — within budget.

## 1 · The acceptance harness

`tests/Acceptance/ExperimentKeyTest.php` runs the real command against four fixtures. Each fixture
directory holds an `expected-context.md` transcribed from the experiment file's step 4 — nothing more
is invented — plus the commit's `diff.patch`.

**Prerequisite: the four diffs are operator-supplied.** The research repository contains no patch
text; the experiments describe commits in a private Laravel + Plaid codebase. This project therefore
ships the expectation files only. Each `diff.patch` must be exported from that codebase by the
operator before the acceptance test can run, and the harness **fails loudly** —
`fixture diff absent — acceptance not run` — rather than passing or skipping quietly. Fabricating a
diff would grade the tool against invented ground truth, which is the one thing the answer-key
method exists to prevent (ADR-001).

| Fixture | Expected bundle (from the experiment file) | Checks |
|---|---|---|
| `experiment-01` | `use` block + `syncFromResponse` body; the single caller of `syncFromResponse`; the `PlaidAccount` model surface (`forItem`, `official_name`, fillable/casts); `upsertFromPlaid` from the same repository file. Transaction premise **flagged**. | Recall of all four; missing-import assertion present; transaction item has `lever: flagged` |
| `experiment-02` | **Almost empty.** At most nothing. | Precision: pulling the `PlaidAccount` model "just in case" **fails** the test |
| `experiment-03` | Cache-store, schema-index, and trashed-data premises **flagged** (X3 — no config/migration fetch in Phase 1); `ExchangePublicTokenRequest` and the route group are fetch-type only if named in the diff | Flag texts present and attributed; no dedicated config resolver invoked |
| `experiment-04` | `PlaidClient::createLinkToken` slice; `PlaidItemStatus` enum slice; call sites of `reactivate(` under `app/` | Reverse-caller item present with call-site provenance; no false extra fetches on the clean parts |

Two properties are asserted for every fixture: **determinism** (two runs, byte-identical output — which is why `filesUnder()` and call sites are lexicographically ordered)
and **budget honesty** (`used_tokens ≤ budget_tokens`, and every omission appears in `dropped[]`).

Recall and precision are *reported by the harness as counts against the expectation file*, not
graded into a pass/fail score by the tool — the tool judges nothing (P6). A human reads the table.

## 2 · Unit tests

`tests/Unit/` mirrors `src/`. All ports are faked in memory (`tests/Fakes/`); no unit test touches
a real filesystem. The tests worth naming up front, because they encode the evidence:

| Test | Asserts |
|---|---|
| `LeverPolicyTest` | The exact decision rule from `fetch-vs-flag.md`: named + single + depth-one ⇒ fetched; reverse-graph-deep or runtime/data ⇒ flagged |
| `OwnFileAssertionExtractorTest` | A symbol used in the region but absent from the `use` block yields a `SameFileSymbolAbsence` — the *absence* case that forward-following cannot see |
| `NamedReferenceResolverTest` | Depth one only: the resolved file's own references produce **no** further assertions |
| `NamedReferenceAssertionExtractorTest` | Only the four recognised reference forms ([01-architecture §3.3](01-architecture.md)) yield assertions; any other form yields none |
| `ChangedSignatureAssertionExtractorTest` | Arity/parameter-shape change detected from old vs new signature; unchanged signatures produce nothing |
| `BundleItemTest` | Constructing an item without a reason or a lever is rejected (P5) |
| `BudgetEnforcerTest` | Over-budget drops follow the `ItemPriority` order stated in [01-architecture §3.4](01-architecture.md), and every drop is recorded (P7) |
| `AssumptionWriterTest` | One statement per catalogue premise; an unknown premise is impossible to construct (A009) |

## 3 · What is *not* tested, on purpose

- Whether the bundle improves a review. Untestable here; it is Phase 2's scored run, and the
  reason Phase 1 exists at all.
- Whether a flag captures as much value as a fetch. Open question in the research; the tool must
  not assume either way.
- Performance. No benchmarks, no thresholds — no optimisation before measurement.

## 4 · First run after build

Per the implementation spec, the first input after the acceptance test passes is the **sloppy
Experiment 5 commit**, which the research repository never ran. If it produces an assertion kind
the four extractors cannot express, the move-set was not bounded and the scope in ADR-003 is
incomplete — which is a finding to record in the research repository, not a patch to make here.
