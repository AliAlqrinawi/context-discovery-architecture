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

**Each expectation entry is marked `fetch-expected` or `flag-satisfied`.** An experiment's step-4
minimum-context list describes what a *human* assembled by hand, so some entries name a source the
tool is not permitted to fetch. The implementation spec settles how to score those: flags count "as a
catch for the flag-type findings (e.g. the transaction assumption)". The expectation files therefore
carry the mark, and the harness compares each entry against it. Without the mark, the acceptance
contract demands output the implementation contract forbids — the two must agree, and the research
says which way.

| Fixture | Expected bundle (from the experiment file) | Checks |
|---|---|---|
| `experiment-01` | `fetch-expected`: `use` block + `syncFromResponse` body; the `PlaidAccount` model surface (`forItem`, `official_name`, fillable/casts); `upsertFromPlaid` from the same repository file. `flag-satisfied`: the caller of `syncFromResponse` — the transaction question, resolved by the `surrounding-transaction` premise, because no signature changed and R3 restricts the caller grep to changed signatures | Recall of all four findings; missing-import assertion present; the transaction item has `lever: flagged`; **no** caller grep was performed |
| `experiment-02` | **Almost empty.** At most nothing. | Precision: pulling the `PlaidAccount` model "just in case" **fails** the test |
| `experiment-03` | `flag-satisfied`: cache-store, schema-index and trashed-data premises (X3 — no config/migration fetch in Phase 1), each fired by its trigger in [ADR-A009](decisions/ADR-A009-premise-catalogue.md). `fetch-expected`: `ExchangePublicTokenRequest` and the route group only if named in the diff | Flag texts present and attributed; no dedicated config resolver invoked |
| `experiment-04` | `fetch-expected`: `PlaidClient::createLinkToken` slice; `PlaidItemStatus` enum slice; call sites of `reactivate(` under `app/` (a real signature change, so the grep runs here) | Reverse-caller item present with call-site provenance; no false extra fetches on the clean parts; **no premise emitted** — this fixture is the precision guard for the transaction trigger |

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
| `UnverifiablePremiseAssertionExtractorTest` | One case per trigger in [ADR-A009](decisions/ADR-A009-premise-catalogue.md): each trigger present ⇒ exactly one premise; each trigger absent ⇒ none. Experiment 2's trace-logging shape yields zero premises |
| `CallerResolverTest` | The grep runs for `ChangedSignature` only; a transaction-style caller question produces a flag and performs no search |

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
