# ADR-A029 · The fourth recognised form option B needs, what it admits, and the three changes that must land as one

- **Status:** Accepted
- **Date:** 2026-09-23
- **Phase:** Phase 6 · the last prerequisite gate for
  [ADR-A027](ADR-A027-inherited-member-silence-gate-step.md)'s option B — the recognised-forms
  table of `01-architecture.md` §3.3. The investigation and the proposal; **not** the row, not the
  code.
- **Implements:** ADR-A003, P3, P6, P10, ADR-A009, ADR-A010, ADR-A018, ADR-A019, ADR-A020,
  ADR-A024, ADR-A027, ADR-A028
- **Changes:** **nothing.** No form, no extractor, no premise, no test, no golden. Two dated
  corrections are appended to ADR-A027 and ADR-A028.

## Why this decision exists

ADR-A027 located the gap at `OwnFileAssertionExtractor.php:85` — a `$this->m(` call to a member
the file does not declare is recognised and then dropped, and nothing downstream ever sees it.
ADR-A028 settled what the statement that would replace the silence must be. Neither can take
effect until an assertion exists, and an assertion exists only if `01-architecture.md` §3.3's
closed list of recognised forms admits the shape. This ADR reads the table, the code and the
tests that enforce it, says exactly what the addition would be and what else it would admit, and
records that the addition cannot land alone.

## 1 · What the table is for

§3.3, in its own words:

> *"R2's recall is defined by this closed list — the **cross-file** forms only. A reference to a
> member of the changed file itself (`$this->method(`, e.g. Exp 1's `upsertFromPlaid`) is a
> `SameFileReference` emitted by `OwnFileAssertionExtractor`: research context type 1, not
> type 2. Each form below is required by a finding; any other form produces **no** assertion —
> no inference, no guessing (ADR-A003)."*

And `NamedReferenceAssertionExtractor`'s header, the same rule in code: *"The list is
deliberately short, and widening it is an architecture change, not a bug fix … The two
extractors partition the same scan and never both fire on one reference."*

What closing it protects: **R2's recall is a definition, not a behaviour.** The engine
recognises three syntactic forms, each traced to a Phase 0 finding (Exp 4, Exp 4, Exp 1), and
everything else is silence by design; Experiment 2's near-empty bundle is guaranteed by the
shortness of the list, not by judgement (P6). Enforced three ways: one private method per form
and no fourth; the negative providers `formsOutsideTheClosedList` (11 lines) and
`formsThatAreNotClassReferences` (8 lines); and `ArchitectureBoundaryTest::testTheMoveSetIsClosed`
locking five extractors and three resolvers.

The silence at E5.4 is pinned by name:
`OwnFileAssertionExtractorTest::testACallToSomethingThatIsNotAMemberOfThisFileYieldsNothing`,
whose comment reads *"Inherited or magic; resolving it would need the parent, which is depth two
(P3)."* It is a case the table's tests assert, not one they missed.

## 2 · The form

| | |
|---|---|
| **Recognised** | `$this` `->` `T_STRING` `(` — the token pattern `siblingCallsIn` already matches at `OwnFileAssertionExtractor.php:163–199` — **and** the name is absent from `MemberSlicer::memberNames($fileText)`. Today that second condition is the `continue` at line 85; the form is that branch, kept and routed instead of dropped |
| **Subject** | **the calling class's FQCN**, `{namespace}\{class}::{m}` — `App\Http\Controllers\Admin\MenuPdfController::success`. The region names `$this`, and `$this` is the current class; namespace and class name are single-file facts |
| **Kind** | `named_reference` — the subject encoding *"Fqcn::member"* fits, and the declaration is by definition in another file |
| **Home** | **`NamedReferenceAssertionExtractor`**, as a fourth form — not `OwnFileAssertionExtractor`. The declaration is cross-file, and `testThisExtractorNeverEmitsACrossFileNamedReference` stays true. One consequence: `forRegion`'s gate `if ($imports === []) return [];` **must not apply to this form** — a file with no `use` block can still call an inherited member |
| **Not recognised** | a property read (`$this->success;`); a dynamic call (`$this->$m(`); a call on a local or another object (`$x->m(`; `$this->prop->m(` is form 2); a member the file declares (stays `SameFileReference`); `parent::`/`self::`/`static::` (§4); a closure's `$this` — indistinguishable by tokens from the enclosing class, a stated imprecision rather than a fail-closed case |

**Why the calling class and not the parent.** Choosing the parent as subject pre-resolves one
hop *inside extraction*, which is resolution's job, and it is wrong for the own-trait case:
`PersonalityResource::resolveLocale` has parent `JsonResource`, and the member is in
`ResolvesLocale`, a trait the class itself uses. The parent is a fact the walk reads; the
subject is what the region names. **This narrows ADR-A028's "calling class or its parent" to the
calling class**, and ADR-A028 §8 and the backend's subject-matching question should be read with
that narrowing.

## 3 · `extends Name` is not needed and should not be added

ADR-A027 §9 and ADR-A028 §9 listed the `extends Name` *position* as a prerequisite. That was
the author's error, corrected here: option B needs `extends` and `use <Trait>` **read as facts**
from the changed file's own text during resolution — a file R1 already loads — not
**recognised as a form** that emits an assertion. The walk reads the clause; nothing asserts it.

What the form would have admitted, counted on added lines of the three scored commits:

| Commit | `extends` clauses on added lines | What they name |
|---|---:|---|
| `ec92403` D1 | 2 | `Controller`, `TestCase` |
| `ee5a2e6` | 8 | `Controller` ×2, `BaseFormRequest` ×2, `JsonResource`, `Model`, `Seeder`, `Migration` |
| `407c110` | 14 | **`Migration` ×14, every one an anonymous class**, on a commit whose bundle is currently — and correctly — empty at zero tokens (M32) |
| **total** | **24** | every one on an `OMIT` row (H.12, H.16, M.5) or a dependency settlement |

Dated notes removing `extends Name` as a prerequisite are appended to ADR-A027 and ADR-A028 with
this ADR.

## 4 · What the form admits — shapes 5 and 6

The table cannot see where a member lives. Adding the form admits every undeclared `$this->m(`.
Counted on added lines, distinct `(file, method)`:

| Shape (ADR-A027 §2) | D1 | ee5a2e6 | 407c110 |
|---|---:|---:|---:|
| 2 — own trait (`resolveLocale`) | 0 | 1 | 0 |
| 3 — parent class directly | 0 | 0 | 0 |
| 4 — parent's trait (`success`, `deleted`, `created`) | 2 | 4 | 0 |
| **5 — dependency ancestor** | **8** | **3** | 0 |
| **6 — `parent::` / `self::` / `static::`** | 1 (`parent::setUp()`) | 0 | 0 |
| **admitted by the form** (shapes 2–5) | **10** | **8** | **0** |

**Shape 5 is indistinguishable at the table.** All eight D1 test helpers — `assertSame`,
`get`, `postJson`, `deleteJson`, `assertCount`, `assertTrue`, `assertNotSame`,
`assertGreaterThan` — have `Tests\TestCase`, **a project class**, as their immediate parent. The
form emits `Tests\TestCase::assertSame` exactly as it emits `MenuPdfController::success`; only
the walk discovers that the ancestry leaves project code two files later. **S2's cost is
therefore unavoidable at extraction and can only be shaped downstream.** That turns ADR-A028's
S1/S2 split from a preference into an argument: the table cannot filter shape 5, so the only
place its cost can be kept off the bundle is the lever — S1 an item, S2 a diagnostic. This ADR
records the split as **argued for**, still unmeasured.

**Shape 6 stays out.** `parent::`, `self::`, `static::` are excluded by `NOT_CLASS_NAMES`
(line 47) and pinned by `formsThatAreNotClassReferences`; the proposed form matches `$this ->`,
not `T_STRING ::`, and does not touch them. One site on three commits; no key has asked for it;
`parent::__construct()` is its commonest instance, which no reviewer needs. Admitting it would
need its own row and its own case.

## 5 · Interaction with the S1/S2 split

§3.3 does not need to know the distinction; it is downstream, and the resolver decides:

| Component | Decides |
|---|---|
| `LeverPolicy` | placeability of the class — the calling class is always placeable (it is the changed file), so `Fetched`; unchanged |
| `NamedReferenceResolver`, with the D2 walk | S1 (found in a project ancestor) or S2 (the ancestry leaves project code first); where `extends` / `use` are read as facts |
| `DiscoverContext` 5c / 5d | S1: a flag with its premise named explicitly, the `call-sites-truncated` pattern through `statementForPremise`; S2: the settled-negative diagnostic path |
| `AssumptionWriter` | renders S1's template |

## 6 · Three changes must land together, not two

A recognised form landing alone collapses as follows, traced on `v0.2.0`, for
`MenuPdfController::success`:

1. `LeverPolicy` — the calling class is placeable (its own file) → **`Fetched`**.
2. `NamedReferenceResolver` opens the placed file — the changed file itself — and finds no
   `success` → **member not found**.
3. `DiscoverContext` 5c fires today's flag: *"named reference could not be resolved on disk;
   contract unverified"* → **a false sentence**; it is on disk, in `ApiResponse.php`.
4. 5c-i: ADR-A020's fallback — *placeable + project + member unresolved*, all three hold → the
   **surface of the calling class is fetched**, and for a new file that is the changed file's
   own class.
5. **The diff is duplicated into the bundle**, which ADR-A018 (a created file is input, not
   context) and ADR-A019 (context the diff already shows is not fetched) prohibit.

So the form (§3.3), the premises and the D2 walk (ADR-A009, ADR-A010), **and a guard that
ADR-A020's surface fallback never targets the assertion's own origin file** are one change with
three gates. Without the guard, S1's flag would be right and the surface beside it would be the
diff. The guard is its own decision on ADR-A020, and **it is the next gate.**

## 7 · The golden goes stale silently

Committed baselines, and what the form does to them:

| Baseline | Read how | Effect |
|---|---|---|
| `laravel-m1/baseline-v0.1.0.json` | regenerated by `LaravelFixtureBaselineTest` on every run | **unchanged** — none of the ten scenario diffs contains an undeclared `$this->m(` |
| `golden/m26-bundle.v2.json` | **read from disk** by `BundleSchemaConformanceTest`: schema-validated, then asserted at 2 assertions / 22 items / 562 tokens | **goes stale.** The M26 diff contains `return $this->success(new BranchResource(...), ...)`; a fresh run would carry a third assertion. The test reads the file and never regenerates it, so **the test stays green while the golden no longer matches the engine — the worst kind of pass**, a fixture that certifies a shape the engine no longer produces |
| `experiment-05..29` captured bundles | evidence, never regenerated (ADR-A024) | unchanged by design; comparison across the boundary means re-running |

A **new golden**, regenerated under the changed engine, and a **`POLICY` bump** — the premise
catalogue moves, so `PolicyVersionGuardTest` fails until the constant does — are required with
the change, not after it.

## 8 · Prerequisites and test inventory

**Prerequisites, all gated, none cleared here:**

| What | Gate |
|---|---|
| A fourth row in §3.3's table, and its prose *"A reference to a member of the changed file itself (`$this->method(`) is a `SameFileReference`"* narrowed to *a member the file declares* | `01-architecture.md` §3.3 — the closed interface this ADR is about |
| Two catalogue premises with fixed statements, S1 and S2 | ADR-A009 (ADR-A028 §9) |
| The verify-only D2 walk through project files | ADR-A010 D2 (ADR-A028 §9) |
| A templated flag payload | ADR-A024 (ADR-A028 §7) |
| **ADR-A020's surface fallback excluded from the assertion's own origin file** | **ADR-A020 — the next gate** |
| A new M26 golden and a `POLICY` bump | ADR-A024 / `PolicyVersionGuardTest` |
| A subject-matching rule in the scorer, settled before scoring | backend ADR-B002 |
| `extends Name` removed as a prerequisite | appended to ADR-A027 and ADR-A028 with this ADR |

**Test inventory — engine tests the change would touch:**

| Test | Today | Under the change |
|---|---|---|
| `OwnFileAssertionExtractorTest::testACallToSomethingThatIsNotAMemberOfThisFileYieldsNothing` | asserts the exact silence | inverted: becomes the positive case, in the other extractor's test |
| `NamedReferenceAssertionExtractorTest::formsOutsideTheClosedList` → `'a same-file sibling belongs to the own-file move'` | `$this->upsertFromPlaid(` yields nothing here | stays true for a **declared** sibling; the undeclared case added as a form-four positive |
| `OwnFileAssertionExtractorTest::testThisExtractorNeverEmitsACrossFileNamedReference` | holds | holds, because the form lives in `NamedReferenceAssertionExtractor` |
| `NamedReferenceAssertionExtractorTest::testAFileWithNoImportsYieldsNothing` | holds | **must be narrowed**: forms 1–3 still yield nothing without imports; form 4 must not be gated by `$imports === []` |
| `formsThatAreNotClassReferences` (`self`, `static`, `parent`) | holds | holds — shape 6 stays out |
| `testAPropertyReadIsNotASiblingCall` | holds | holds |
| `ArchitectureBoundaryTest::testTheMoveSetIsClosed` — five extractors, three resolvers | holds | holds — no sixth extractor, no fourth resolver |
| the six-kinds lock | holds | holds — `named_reference`, no new kind |
| `AssumptionWriterTest::assertCount(7, …)` | holds | 9 |
| `PolicyVersionGuardTest` | holds | fails until `POLICY` moves from `2` to `3` |
| `BundleSchemaConformanceTest` golden assertions (2 / 22 / 562) | holds | **holds on the stale file** — see §7; a new golden is required |
| `LaravelFixtureBaselineTest` | holds | holds — no scenario contains the form |

No sixth extractor. No new `AssertionKind`. No new lever.

## Related

- [ADR-A027](ADR-A027-inherited-member-silence-gate-step.md), [ADR-A028](ADR-A028-inherited-member-statement-gate-step.md)
  — the gap and the statement; both corrected by appended note on `extends Name`.
- [ADR-A020](ADR-A020-the-project-class-surface-fallback.md) — the fallback that would duplicate
  the diff; the next gate.
- [ADR-A018](ADR-A018-a-created-file-is-input-not-context.md), [ADR-A019](ADR-A019-context-the-diff-already-shows-is-not-fetched.md)
  — what step 5 of §6's chain violates.
- [ADR-A024](ADR-A024-bundle-contract-v2.md) — the golden, the policy version.
- `01-architecture.md` §3.3; implementation repository
  `src/Discovery/Extraction/NamedReferenceAssertionExtractor.php`,
  `src/Discovery/Extraction/OwnFileAssertionExtractor.php:85`,
  `tests/Unit/Discovery/Extraction/*Test.php`, `tests/Acceptance/fixtures/golden/`.

---

## Correction · 2026-09-24 · §6's chain ends differently, and three changes are two

**Source:** the ADR-A020 investigation and addendum of this date. The text above is unaltered.

§6 step 5 says the fallback would fetch *"the changed file's own class … the diff duplicated,
which ADR-A018/A019 prohibit."* For a **created** file that is already blocked: `DiscoverContext`
5c-i passes every fallback slice through `Diff::showsEntirely`, which returns true for any span
in a new file, so nothing survives to become an item. The real exposure is a **modified** file:
`surface()` slices every member the file declares, `showsEntirely` filters only those inside a
changed region, and the fallback would fetch every *unchanged* sibling of the calling class — not
the diff, but the rest of the file, which the region never called and no reviewer asked for.
**The chain's conclusion stands; its stated harm does not.**

With ADR-A020's fourth condition landed alone (its addendum, and the engine commit it names),
§6's *three changes must land together* becomes **two**: the form (§3.3) and the premises with
the D2 walk (ADR-A009, ADR-A010). The guard is in place before either, and its synthetic test
pins the invariant they will inherit.
