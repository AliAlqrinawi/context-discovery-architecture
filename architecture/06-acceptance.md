# 06 · Acceptance and Test Strategy

The acceptance criterion was stated in `docs/03-phase1/implementation-spec.md`: Phase 1 is done
when, for all four Phase 0 commits, the tool's bundle reproduces the **minimum-context list** in
that commit's experiment file. That criterion binds the tool to four inputs this repository never
had — the commits live in a private codebase and the research documents carry no patch text — so
the harness built for it failed four tests on every run from M10 to M27 and was **retired, not
faked**, by [ADR-A025](decisions/ADR-A025-the-phase-0-gate-is-retired-not-faked.md).

## 1 · The acceptance strategy

Three kinds of test carry the acceptance load now, each of which actually runs:

| Kind | Where | What it grades |
|---|---|---|
| **Keyed synthetic fixtures** | `experiment-14`, `-15`, `-16` under `tests/Acceptance/fixtures/`, each with a hand-written `answer-key.json`, a `diff.patch` and a checked-in post-image `repo/` — run by `ProjectSurfaceFallbackAcceptanceTest`, `OriginDoesNotChangeOutcomeTest`, `DuplicateSliceIdentityTest` | the whole command against a key written **before** the run (ADR-001), on inputs the repository holds |
| **The Laravel baseline** | `laravel-m1`: ten scenario diffs against three repository variants, with `answer-key.json` and a recorded `baseline-v0.1.0.json` — run by `LaravelFixtureBaselineTest` | what the tool *should* produce and what it *does* produce, kept as two checkable things; byte-identical determinism across runs |
| **The golden bundle** | `fixtures/golden/m26.diff` + `m26-bundle.v2.json` — run by `BundleSchemaConformanceTest` | the published contract: schema conformance of real output, the closed sets both ways, and a recorded end-to-end artefact whose input is committed beside it |

The Phase 0 answer keys are preserved as documentation at `docs/research/phase0-keys/` in the CLI
repository. The claims they make are asserted by named tests — every premise, the zero-call-site
negative, changed-signature call sites, named references — and the one claim that had no home
elsewhere, experiment 02's *"pulling the model just in case is a precision failure"*, is
`SelfContainedDiffTest`. The exit-code contract the harness also carried is `ProcessContractTest`.

Two properties are asserted for every fixture that runs: **determinism** (two runs, byte-identical
output on both streams — which is why `filesUnder()` and call sites are lexicographically ordered)
and **budget honesty** (`used_tokens ≤ budget_tokens`, and every omission appears in `dropped[]`).

Recall and precision are *reported as counts against a key*, not graded into a pass/fail score by
the tool — the tool judges nothing (P6). A human reads the table.

## 2 · Unit tests

`tests/Unit/` mirrors `src/`. All ports are faked in memory (`tests/Fakes/`); no unit test touches
a real filesystem. The tests worth naming up front, because they encode the evidence:

| Test | Asserts |
|---|---|
| `LeverPolicyTest` | The exact decision rule from `fetch-vs-flag.md`: named + single + depth-one ⇒ fetched; reverse-graph-deep or runtime/data ⇒ flagged |
| `OwnFileAssertionExtractorTest` | A symbol used in the region but absent from the `use` block yields a `SameFileSymbolAbsence` — the *absence* case that forward-following cannot see |
| `NamedReferenceResolverTest` | Depth one only: the resolved file's own references produce **no** further assertions |
| `NamedReferenceAssertionExtractorTest` | Only the three recognised **cross-file** reference forms ([01-architecture §3.3](01-architecture.md)) yield assertions; any other form yields none. A `$this->method(` sibling is a `SameFileReference` and must **not** appear here (freeze review 04) |
| `ChangedSignatureAssertionExtractorTest` | Arity/parameter-shape change detected from old vs new signature; unchanged signatures produce nothing |
| `ChangedReturnContractAssertionExtractorTest` | All four conditions of [ADR-A023](decisions/ADR-A023-changed-return-contract.md) must hold together; each negative control removes exactly one. An added return the region's span does not contain yields nothing |
| `ChangedReturnContractHunkOffsetTest` | Driven by real `git diff` output: the member named is the one containing the changed return, whether the hunk opens between two members or inside the previous one |
| `BundleItemTest` | Constructing an item without a reason or a lever is rejected (P5) |
| `BudgetEnforcerTest` | Over-budget drops follow the `ItemPriority` order stated in [01-architecture §3.4](01-architecture.md), banding from the assertion's `kind` + the item's `lever` alone; every drop is recorded (P7) |
| `BundleSchemaConformanceTest` | `schema/bundle-v2.schema.json` is the single source: every kind, lever and diagnostic type is asserted **both ways** against the code, the fixed order is asserted against `BundleAssembler`, and real output plus the golden M26 fixture are validated against the schema ([ADR-A024](decisions/ADR-A024-bundle-contract-v2.md)) |
| `BundleTest` | An item naming an assertion the bundle does not carry is rejected on construction — P5 surviving the reason's move onto the assertion |
| `PolicyVersionGuardTest` | `run.policy_version` is hashed against the classes it claims to describe, so a rule change without a version bump fails |
| `OwnFileAssertionExtractorTest` (second case) | A `$this->method(` sibling yields `SameFileReference`, never `NamedReference`; a cross-file class yields `NamedReference`, never `SameFileReference` |
| `AssumptionWriterTest` | One statement per catalogue premise; an unknown premise is impossible to construct (A009) |
| `UnverifiablePremiseAssertionExtractorTest` | One case per trigger in [ADR-A009](decisions/ADR-A009-premise-catalogue.md): each trigger present ⇒ exactly one premise; each trigger absent ⇒ none. Experiment 2's trace-logging shape yields zero premises |
| `CallerResolverTest` | The grep runs for `ChangedSignature` and `ChangedReturnContract` only; a transaction-style caller question produces a flag and performs no search. A search completing with **zero** call sites yields no item and no flag — one stderr diagnostic (freeze review 05) — while an unreadable or absent scope yields a `caller-search-failed` flag, never `unresolved-reference` (freeze review 06) |

## 3 · What is *not* tested, on purpose

- Whether the bundle improves a review. Untestable here; it is Phase 2's scored run, and the
  reason Phase 1 exists at all.
- Whether a flag captures as much value as a fetch. Open question in the research; the tool must
  not assume either way.
- Performance. No benchmarks, no thresholds — no optimisation before measurement.

## 4 · First run after build

Per the implementation spec, the first input after the acceptance test passes is the **sloppy
Experiment 5 commit**, which the research repository never ran. If it produces an assertion kind
the five extractors cannot express, the move-set was not bounded and the scope in ADR-003 is
incomplete — which is a finding to record in the research repository, not a patch to make here.
