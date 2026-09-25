# ADR-A028 · The statement option B needs is an item, not a diagnostic — and what that costs

- **Status:** Accepted
- **Date:** 2026-09-23
- **Phase:** Phase 6 · [ADR-A009](ADR-A009-premise-catalogue.md)'s gate, opened for the fixed
  statement [ADR-A027](ADR-A027-inherited-member-silence-gate-step.md)'s option B needs. The
  investigation and the proposal; **not** the catalogue entry, not the statement, not the code.
- **Implements:** ADR-A003, ADR-A009, P6, P10, ADR-A010, ADR-A013, ADR-A024, ADR-A027
- **Changes:** **nothing.** No premise, no statement, no lever, no schema, no production code.
  One dated correction is appended to ADR-A027.

## Why this decision exists

ADR-A027's appendix made option B the candidate: walk `extends` and `use <Trait>` through
project files to a fixed point, then state where the inherited member is declared, without
fetching it. It described that outcome as costing zero tokens. Whether it costs zero depends on
a question ADR-A027 did not ask: is the statement a **premise** (a flag, an item, tokens), or a
**diagnostic** (a stderr line, no item, free)? The audience decides it, and the audience is on
record: six of seven reviewers wrote `Controller.php::<member>` when `Controller.php` declares
nothing. The statement's job is to correct that error. This ADR settles which shape can.

## 1 · Item, not diagnostic

The engine has exactly two outcomes for a reference it will not fetch, traced on `v0.2.0`:

| Path | Where | What the reviewer gets | Tokens |
|---|---|---|---|
| **Flag** — `DiscoverContext` 5b/5c | an item, `lever: flagged`, payload one fixed sentence from `AssumptionWriter::STATEMENTS`, premise from `PremiseCatalogue` | in the bundle | ~20 |
| **Settled negative** — 5d, AA11 | one stderr line (*"dependency member: … declared at …; source not fetched"*), **no item** | not in the bundle, and not in `diagnostics[]` either: ADR-A024's closed enum mirrors only `call_sites_truncated` and `post_image_contract_violated` | 0 |

ADR-A024 states the audience rule: *"the flag is context an agent reads inside the bundle it was
given … `diagnostics[]` is for a consumer that wants values."* The bundle is what the agent acts
on. And the record tests the alternative directly: the M17-protocol packets shipped stderr as
`context-diagnostics.txt`, so every DIFF_PLUS_BUNDLE reviewer had the diagnostic stream in
front of them - and **six of seven reviewers who named this path wrote `Controller.php::<member>`
with that file in hand and were not corrected.** A stderr line is the shape that reaches the
harness. It is not the shape that reaches the reviewer.

So the statement **must be an item**, and an item can only be a flag: fetching is closed by
[ADR-A010 §4](ADR-A010-inheritance-and-annotation-traversal.md), and the lever enum is
`fetched | flagged`. That forces ADR-A009's own condition - a premise exists only where something
is **unverified**, and a settled question gets none (freeze review 05 rejected a seventh premise
on exactly that ground). Something is unverified here: the member's **contract**. Its body is
located and deliberately not read; the reviewer's reading of what `success()` returns is an
assumption. That is the same premise `unresolved-reference` already states - *"contract
unverified"* - with a true location clause where today's sentence has a false one. ADR-A013 chose
the diagnostic for dependency members on Experiment 2's ground, *the reviewer already has it*;
7 of 45 reviewers say they do not have this. Same mechanism, that ground removed.

> **Decision on shape:** a new catalogue premise, the true sibling of `unresolved-reference`,
> emitted as a flagged item. Not a diagnostic.

## 2 · ADR-A027 §4's "zero tokens" is wrong

It was written on the diagnostic assumption. A dated correction is appended to ADR-A027 pointing
here. The body of that ADR is not rewritten.

## 3 · The real cost — estimates

Per distinct `(file, method)` pair on added lines (ADR-A021: flags are keyed by origin, so two
calls in one region collapse to one). Token figures are ⌈bytes / 4⌉, the engine's own
`TokenEstimate` rule, on the statements in §5. **Estimates, every figure.**

| | D1 `ec92403` (606-token bundle) | ee5a2e6 (1078-token bundle) |
|---|---|---|
| S1 items - declared in a project ancestor | 2 (`success`, `deleted` → E5.4) ≈ **120 tokens** | 5 (`success` ×2 files, `created`, `deleted` → H.3; `resolveLocale` → H.4) ≈ **260 tokens** |
| S2 items - ancestry enters a dependency | 8 test helpers ≈ **385 tokens** | 3 test helpers ≈ **145 tokens** |
| **total** | **+10 items, ≈ 505 tokens** | **+8 items, ≈ 405 tokens** |
| on the key's OMIT rows | 8 of 10 (E5.12) | 3 of 8 (H.19) |
| under the key's FETCH rows | 2, as flags - scored `noise` by lever, carrying the citation seven reviewers asked for | 5, the same |

Option B as an item roughly doubles D1's bundle and adds a third to ee5a2e6's - and on both
commits **S2 dominates**: the test-helper population pays 385 tokens on D1 to say eleven true
things no reviewer asked for.

## 4 · The candidate: split the two statements by lever

| | Statement | Shape | Ground |
|---|---|---|---|
| **S1** | declared in a project ancestor, cited, body not fetched | **flagged item** | 7 of 45 reviewers asked for exactly this; the bundle is where they would see it |
| **S2** | ancestry leaves project code without finding the member | **settled-negative diagnostic**, no item | no evidence anyone needs it; P10 is satisfied because the *boundary* is stated on stderr - what was walked, where it stopped - which is not silence |

The reasoning for preferring the split, recorded so it can be tested rather than assumed: the
evidence that reviewers need a statement is evidence about S1's case only - every one of the
seven named a member that S1 would find. S2's case is the test-helper population, which every
key so far marks OMIT and which no reviewer named. Paying S2's tokens as items would spend twice
S1's cost on the case with no evidence behind it, and would turn a candidate with a defensible
cost into one that doubles a bundle. The alternative to the split - option D, settling the test
helpers through the framework table - is ADR-A011-gated and would remove S2's population by a
different route; either can be tested.

**It remains to be measured.** The split is a candidate for ADR-A027 §7's experiment, scored
against two authors' keys; it is not a decision, and this ADR does not make it one.

## 5 · What the statements say

Two constraints pull against each other. ADR-A009's statements are literal constants - there is
no interpolation anywhere in `STATEMENTS` - and the ADR rejected composed sentences as
*"non-deterministic in practice and unbounded in scope"*. But this statement's job is to name a
file, and a constant cannot. The templates below interpolate only facts the walk checked.

**S1 template:**

> `ASSUMPTION: {member}() is not declared in {class}{, or in its parent {parent}}; it is declared in {trait|parent} {declaring FQCN} at {path}:{line}{, used by {applier}}; body not fetched, contract unverified`

"Not declared in its parent" appears only when a parent was walked and did not declare the
member. It is the clause that corrects the reviewers' error, and it is never asserted about a
file the walk did not open.

Rendered, with the engine's token estimate:

| Case | Rendered | Tokens |
|---|---|---:|
| `ApiResponse::success` via `Controller` — two hops, trait on the parent | `ASSUMPTION: success() is not declared in MenuPdfController or in its parent App\Http\Controllers\Controller; it is declared in trait App\Traits\ApiResponse at app/Traits/ApiResponse.php:9, used by that parent; body not fetched, contract unverified` | 61 |
| `ResolvesLocale::resolveLocale` — one hop, trait on the class | `ASSUMPTION: resolveLocale() is not declared in PersonalityResource; it is declared in trait App\Http\Resources\Concerns\ResolvesLocale at app/Http/Resources/Concerns/ResolvesLocale.php:7, used by the class itself; body not fetched, contract unverified` | ~50 |
| a member declared on the parent class directly | `ASSUMPTION: run() is not declared in OneLevel; it is declared in parent App\Support\Direct at app/Support/Direct.php:12; body not fetched, contract unverified` | ~40 |

**S2 is a second statement**, because S1 would be false when nothing was found, and today's
`unresolved-reference` sentence would be false too - the reference *was* resolved as far as
project code goes. Touching `unresolved-reference` itself would change what every scored run
since M17 compares, so S2 is new, not an edit:

| Case | Rendered | Tokens |
|---|---|---:|
| the walk reaches a dependency without finding the member | `ASSUMPTION: getJson() is not declared in MenuPdfTest or in its project ancestor Tests\TestCase; its ancestry continues into Illuminate\Foundation\Testing\TestCase, which was not walked; contract unverified` | ~48 |

Under §4's split, S2 is emitted as a diagnostic and its token figure is zero.

## 6 · When it does not fire — fail-closed

| Condition | Behaviour |
|---|---|
| `extends` names a class the map cannot place | walk stops; **S2** with "continues into {name}, which could not be placed" — never S1 |
| a file on the chain is unreadable | walk stops; S2 with "unreadable at {path}"; nothing fetched, nothing guessed |
| the member is declared in **two** ancestors (two traits on one class, or a trait and a parent) | **S1 does not fire.** PHP resolves this by `insteadof` or by class-over-trait precedence; reproducing that is inference (P6). Fall to S2 with "declared in more than one ancestor; not disambiguated" |
| cyclic `use`/`extends` (impossible in valid PHP, possible in a malformed tree) | visited-set; the walk stops on revisit and falls to S2 |
| the member is declared `abstract` in the ancestor | S1 fires — an abstract declaration *is* the declared contract, and the citation lands on it |
| the walk would enter `vendor/` | it does not; **no dependency file is opened on the item path** (ADR-A012's guarantee holds); S2 names the dependency class by FQCN from the `extends` clause alone |
| the extends/use clause is in the changed file's context lines but the method call is not in the changed region | nothing — cause A (extraction) is what gates entry, and this ADR does not change it |

Bound: project files only, one visited-set, no hop count invented — ADR-A010 rejected an
invented bound as AA1-shaped, and "to a fixed point" is the honest form it named.

## 7 · The templated payload — an argument, not a clearance

`items[].payload` is defined for a flag as *"the assumption sentence and nothing else"*, and
every statement in `STATEMENTS` is a constant. A sentence with an interpolated path and line is
a change to what that field means for `lever: flagged` - ADR-A024's compatibility table names
that the loudest kind of change, the one no validator can catch.

The argument that it should be allowed, recorded as an argument: ADR-A009 rejected composed
flags because they are *"non-deterministic in practice and unbounded in scope"*. A template
filled only from facts the walk checked - a member name, a class name, a path, a line, each read
from a file - is deterministic (P8: same tree, same bytes) and bounded (the slots are fixed, the
values are file contents). The diagnostics already interpolate exactly this tuple on stderr,
deterministically, and have since M4. The *reason* for the rule does not apply to such a
template. The *letter* does, and the letter is a closed interface.

**ADR-A024 is the gate this argument does not clear.** Whether a templated flag payload is an
additive change, a change of meaning, or a new payload kind (the shape ADR-A013 rejected for
"fetch only a signature") is ADR-A024's question, decided there on its own terms. This ADR
records the argument and does not decide it.

## 8 · The lever

**`Flagged`.** `LeverPolicy` has no category for "verified, deliberately not fetched"; the enum is
`fetched | flagged`, closed by the schema and the conformance test, and the policy chooses by
placeability *before* resolution. ADR-A013 is the same mechanism - positive evidence the member
exists, cited to file and line, body withheld - with the opposite outcome, chosen because the
reviewer already knows `Str::slug`. A third lever would be the cleanest expression and the
largest change; it is not needed: an unverified contract is what `Flagged` already means.

One mechanical consequence: `AssumptionWriter::premiseFor` derives the premise from the
assertion's *kind*, so a `NamedReference` failure is always `unresolved-reference`. The
ancestor outcome is known only inside the resolver, so the pipeline would name the premise
explicitly, as it already does for `call-sites-truncated` through `statementForPremise`.
Internal, and it is where the branch would live.

## 9 · Prerequisites

| What option B would need | Gate |
|---|---|
| Two new catalogue premises, S1 and S2, each with its own fixed statement | **ADR-A009** — the catalogue enum, `STATEMENTS`, ADR-A009's trigger table; `AssumptionWriterTest::assertCount(7, …)`; `PolicyVersionGuardTest`, so `POLICY` moves from `2` to `3` and scored runs across the boundary are compared knowingly |
| A **templated** flag payload (§7) | **ADR-A024** — a closed interface; the loudest row of its table |
| `$this->m(` to a member the file does not declare, and `extends Name`, as recognised forms | `01-architecture.md` §3.3 — a closed interface (named in ADR-A027) |
| A resolver that walks `extends` / `use <Trait>` through project files to a fixed point | **ADR-A010 D2** — the verify-only relaxation it left pending an experiment |
| If S2 is a diagnostic and mirrored into the bundle: a new `diagnostics[].type` | ADR-A024, additive |
| If either statement is to stop inside a dependency file to check the member (the ADR-A013 move) | not proposed; the walk stays out of `vendor/` |
| A subject-matching rule in the backend's scorer, before any scoring of option B | backend **ADR-B002** — keys name the declaring site, an S1 assertion names the calling class or its parent; without the rule every S1 scores unkeyed |
| The correction to ADR-A027 §4 | appended with this ADR |

No new `AssertionKind`. No new lever. No framework table entry.

## Related

- [ADR-A027](ADR-A027-inherited-member-silence-gate-step.md) — the gap, the options, the
  pre-check, and the appendix that made option B the candidate.
- [ADR-A009](ADR-A009-premise-catalogue.md) — the catalogue's condition for a premise and the
  fixed-statement rule.
- [ADR-A013](ADR-A013-dependency-members-are-settled-not-fetched.md) — the same mechanism with
  the diagnostic outcome, and the ground that chose it.
- [ADR-A024](ADR-A024-bundle-contract-v2.md) — `items[].payload`'s definition and the
  compatibility table §7 does not clear.
- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) — §4's fetch ban and D2's pending
  verify-only relaxation.
- Implementation repository: `src/Discovery/Flagging/AssumptionWriter.php`,
  `src/Pipeline/DiscoverContext.php` (5b, 5c, 5d, `namedReferenceSettlement`),
  `src/Assembly/TokenEstimate.php`.

---

## Correction · 2026-09-23 · `extends Name` is not a prerequisite; the subject is the calling class

**Source:** [ADR-A029](ADR-A029-recognised-forms-gate-step.md) §2–3. The text above is unaltered.

§9's third row lists the `extends Name` position among the recognised forms option B needs. It is
not one: the walk reads `extends` and `use <Trait>` as facts from the changed file during
resolution; nothing asserts the clause. Removed as a prerequisite.

§8's "calling class or its parent" is narrowed: the subject of the new form is **the calling
class's FQCN** — `App\Http\Controllers\Admin\MenuPdfController::success` — never the parent.
Choosing the parent would pre-resolve a hop inside extraction and be wrong for the own-trait case
(`PersonalityResource::resolveLocale`, whose parent is `JsonResource` and whose member is in
`ResolvesLocale`). The backend's subject-matching question (ADR-B002) should be read with this
narrowing: a key that lists the calling-class subject beside the declaring site is sufficient.

---

## Correction · 2026-09-26 · §5's illustrative S2 sentence does not match the built walk

**Source:** the option-B runs on `ec92403` and `ee5a2e6` (engine `6a77cdb`). The text above is
unaltered.

§5 illustrates S2 as *"getJson() is not declared in MenuPdfTest or in its project ancestor
Tests\TestCase; its ancestry continues into Illuminate\Foundation\Testing\TestCase, which was
not walked"*. That sentence assumes the walk reaches the parent chain. It does not, for these test
classes: each `use`s `Illuminate\Foundation\Testing\RefreshDatabase`, a **dependency trait on the
class itself**, and §6's rule that a member declared in a trait and a parent is not disambiguated
(P6) makes a dependency trait a boundary *before* any parent is opened — whether the trait
declares the member cannot be known without opening it. So the walk stops at hop zero and the
stderr line reads `inherited member unresolved: Tests\Feature\MenuPdfTest::getJson; walked
nothing; continues into a dependency, which was not walked
(Illuminate\Foundation\Testing\RefreshDatabase)`; in the no-vendor run the reason is `names an
ancestor the map cannot place`, same stop, same trait.

The outcome is the one §4 chose — S2, a settled negative on stderr, no item, zero tokens — and
`Tests\TestCase` is never asserted about, which is the property §5 was protecting. What was wrong
is the illustration's assumed stop point, not the rule. The behaviour is tested
(`AncestryResolverTest::testTheWalkStopsAtADependencyTraitEvenWhenAProjectParentMightDeclareTheMember`).

