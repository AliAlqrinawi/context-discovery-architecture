# ADR-A010 · Resolution reads one file beyond the changed file — inheritance and annotation chains are not followed

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 architecture · resolves open question OQ1 raised by the M0 framework-knowledge research
- **Implements:** P3, P4, P9, X1, X2, ADR-002, ADR-A003, ADR-A005
- **Changes:** no module, no port, no capability, no contract. This ADR *classifies* an existing
  guarantee and records an under-build; it adds nothing to the build.

## Why this decision exists

The M0 research (`docs/research/M0-laravel-framework-knowledge.md` in the implementation repository)
measured that on real Laravel code every `named_reference` flag the tool produced was a false
positive, and that a large share of them are members the application's own class does not declare
but *inherits*. Resolving them requires reading a **second** file beyond the changed file: the
parent class, a trait, or the target of a `@mixin` annotation.

Whether that is permitted was left open as **OQ1**, deliberately, because the answer is not a coding
choice — it decides whether P3's "depth one" is a statement about *fetched context* or about *files
opened*. M1 then built the acceptance fixture that makes the question concrete
(`tests/Acceptance/fixtures/laravel-m1/`), with **S05** as the minimal case: one real declared
method, one `extends`, and no Laravel semantics of any kind.

Nothing can be built on top of framework knowledge until OQ1 is settled, so it is settled here.

## The minimal decision case

`App\Models\Package::query()` — fixture scenario **S05**, variant A *and* variant B.

- `app/Models/Package.php` declares exactly two members: `features()` and `scopeActive()`.
  `query` is **not** among them (verified).
- `Illuminate\Database\Eloquent\Model` declares `public static function query()` at `Model.php:1588`
  (verified in the fixture's own vendor tree, `laravel/framework` v12.64.0).
- `class Package extends Model`. PHP resolves `Package::query()` to the parent's declaration. No
  `__callStatic`, no facade, no builder, no annotation, no Laravel semantics — **the plain language
  rule**.
- v0.1.0 emits `ASSUMPTION: named reference could not be resolved on disk; contract unverified`.

The statement is false: the contract *is* on disk, in a file the tool chose not to open. S05 is
therefore the cleanest possible statement of OQ1 — if a traversal is ever justified, it is justified
here first.

## The distinction that had to be made first

The brief asks OQ1 to distinguish *resolving an existing assertion* from *creating a new assertion*.
That distinction is real, and it shows that the frozen "depth one" guarantee is **two separate
guarantees** that have been carried as one phrase:

| | Guarantee | Stated at | Status |
|---|---|---|---|
| **D1** | **Resolved sources never become new input.** Stage 5 never re-enters stage 3; no assertion is ever created from a file that resolution opened | `01-architecture.md` §4 — *"Stage 5 never re-enters stage 3. Resolved sources are payload, never new input."* `05-traceability.md` §5 — *"no queue or worklist type exists to hold pending references"* | **Inviolable.** It is P4/X1, which the research marks **never** — disproven, not merely unjustified |
| **D2** | **Resolution opens at most one file beyond the changed file.** | P3 — *"Resolution stops at depth one … it does not then resolve their references"*; X2 | **An under-build.** The research marks X2 **not yet**, and ADR-002 bans it because *"the reframing removes its justification"* — an absence of motive, not a demonstrated harm |

An ancestry walk that resolves `Package::query` to `Model::query` creates **no new assertion**: the
subject stays `App\Models\Package::query`, the bundle item's `reason` is unchanged, and nothing is
handed back to extraction. **It does not violate D1.** It violates **D2**, and only D2.

This matters because D1 and D2 have different standing. Conflating them would have made OQ1 look
like a question about X1 — which is closed forever — when it is a question about X2, which the
research explicitly leaves open to evidence. Recording the split is the durable part of this ADR;
the decision below is the part a future experiment may reverse.

## Decision

### 1 · `extends` and `use <Trait>` — **disallowed**, as a bounded under-build

Resolution opens the file the PSR-4 map places for the *named* class, slices the named member from
that file's own text, and stops. It does not open the file of a parent class or a trait, whether to
fetch a slice from it or merely to check that a member exists there.

This applies to **both** uses, and the reasons differ:

| Use | Verdict | Why |
|---|---|---|
| **Fetch** a slice from an ancestor into the bundle | **Disallowed** — directly by the frozen documents | This is depth-two *fetched context*, exactly what P3 and X2 name. No experiment's minimum-context list has ever named an ancestor's member; Experiment 1 asks for *"the `PlaidAccount` model"*, not for the model and its base class |
| **Verify only** — open an ancestor, confirm a member name exists, emit nothing | **Disallowed for now** — by precedent and by the rule for gaps, not by an explicit prohibition | No document addresses a read whose only effect is to change a flag's truth value. What the corpus *does* offer is a worked precedent pointing the same way: Experiment 3's `SoftDeletes` finding is the one inheritance-shaped fact in the evidence, and it was resolved from a removed line in the changed file — depth zero, no trait file opened (see *Evidence* below). Beyond that precedent, `evidence-gaps.md` §4 fixes the response to a gap as *an under-build with a named trigger*, and ADR-A003's gate requires *"a new experiment, a requirement entry, an ADR, and a `Wiring` change — four visible steps, in that order."* The experiment is missing |

### 2 · `@mixin` — **disallowed**, on three independent grounds

`@mixin` is treated separately, and the evidence is stronger against it than against `extends`:

1. **It is an unenforced annotation, not a language relation.** `extends` is guaranteed by the PHP
   engine: if `Model.php` declares `query()`, then `Package::query()` provably resolves to it.
   `@mixin \Illuminate\Database\Query\Builder` (`Builder.php:33`) is the author's *claim* about
   dynamic behaviour, which nothing enforces. ADR-A009 requires a flag's fixed statement to be
   *exactly true* of the case it covers; deciding a member exists on the strength of a comment is a
   weaker warrant than the language's own rule, and this ADR does not extend the weaker warrant
   before the stronger one.
2. **It is unreachable without Laravel-specific semantics.** In the only case that motivates it,
   the `@mixin` tag sits on `Illuminate\Database\Eloquent\Builder`, and nothing reaches that class
   except by first applying the Model-to-builder forwarding rule — which is Laravel knowledge, and
   which belongs to a later milestone. Deciding `@mixin` here would decide that milestone's content
   inside this one, on a case the brief scopes as *"zero Laravel-specific semantics"*.
3. **It is strictly deeper.** Measured in the M1 fixture (below), the only `@mixin` case is at four
   file hops. Even if D2 were relaxed to two, it would not reach it.

### 3 · The exact boundary

**Resolution may open exactly one file beyond the changed file: the file the `ClassLocator` places
for the class the assertion names. Depth is counted in files opened, and the bound is one.**

Measured on the M1 fixture (`laravel/framework` v12.64.0), which makes the bound concrete rather
than abstract:

| Fixture row | Subject | Declared at | Files opened beyond the changed file | Last hop is | Verdict |
|---|---|---|---|---|---|
| S07.2 | `Package::active` | `app/Models/Package.php:29` as `scopeActive` | **1** | a naming convention *inside the file already opened* | **within the bound** |
| S01.1 | `Log::info` | `Support/Facades/Log.php:25` as a `@method static` tag | **1** | a docblock *inside the file already opened* | **within the bound** |
| S05.1 | `Package::query` | `Database/Eloquent/Model.php:1588` | **2** | `extends` — language-guaranteed | **outside — blocked** |
| S02.1 | `Package::create` | `Database/Eloquent/Builder.php:1216` | **3** | `__callStatic` forwarding — a Laravel rule | **outside — blocked** |
| S03.1 | `Package::where` | `Database/Eloquent/Builder.php:352` | **3** | as above | **outside — blocked** |
| S03.2 | `Package::orderBy` | `Database/Query/Builder.php:2918` | **4** | `@mixin` — an annotation | **outside — blocked** |

The line falls between rows two and three, and it falls there for a reason that is visible in the
table: rows one and two need **no file the resolver was not already going to open**. That is the
operative test.

> **A recognition rule is admissible under this ADR if and only if every fact it depends on is
> readable in the changed file or in the single file the `ClassLocator` places for the named class.**

Call this **single-file recognition**. It is not a new capability, a new module, or a new port — it
is a constraint on what any future rule may read, expressed so that it can be checked by reading
the rule rather than by running it.

### 4 · What remains forbidden regardless of any future evidence

- **D1**, permanently: nothing a resolver opens may produce an assertion. No worklist, no queue, no
  re-entry into extraction. This is P4/X1, marked **never**.
- **Forward import-following as an organising principle** — following `use` statements outward to
  decide *what to fetch*. ADR-002 records that it would have missed three of Experiment 1's four
  findings.
- **Fetching an ancestor's source into the bundle**, even if D2 is later relaxed for verification.
  Relaxing "may I look?" would not by itself answer "may I include it?", and Experiment 2 is the
  standing constraint on the latter.
- **Reverse-caller as a graph**, config/migration resolvers, severity, and an in-tool LLM — A006,
  X3, X4, X5, all unchanged by this ADR.

## Evidence

### What the frozen corpus says

| Source | Quotation | Bearing |
|---|---|---|
| `architecture-principles.md` P3 | *"Resolution stops at depth one. The engine resolves references the diff names directly; it does not then resolve **their** references."* | The bound, stated in terms of references |
| `architecture-principles.md` P4 | *"The disproven move is banned at the architecture level, not just omitted."* | X1 is a disproof — permanent |
| `requirements.md` X2 | *"nothing in four commits needed it; depth one carried every finding"* | The ban rests on absence of need |
| `00-research-verification.md` §4 | X1 → **never**; X2 → **not yet** | The two are not the same kind of exclusion |
| `ADR-002` "What this decision explicitly bans" | X1 *"as the organising principle"*; X2 *"— the reframing removes its **justification**"* | X2 is unjustified, not harmful |
| `ADR-002` Consequences | *"The engine stays shallow — depth one — because the reframing removes the **motive** to crawl"* | The test for revisiting is whether a motive has appeared **in evidence** |
| `01-architecture.md` §4 | *"Stage 5 never re-enters stage 3. Resolved sources are payload, never new input."* | D1, stated |
| `05-traceability.md` §5 | X2 enforced by *"no queue or worklist type exists to hold pending references"* | The enforcement targets D1's shape; it does not by itself prevent an ancestry walk — which is why this ADR is needed |
| `REVIEW-freeze-01.md` | *"X1–X5 are enforced structurally, not by discipline: … no worklist type that could hold pending references (**depth > 1 is unreachable**)"* | The architecture **asserts** depth>1 is unreachable. An ancestry walk would falsify a stated structural guarantee |
| `evidence-gaps.md` §4 | *"Known under-builds (deliberate, revisit only on new evidence)"*, each with a **trigger to revisit** | The established response to a gap |
| `ADR-A003` | *"Adding a move requires: a new experiment, a requirement entry, an ADR, and a `Wiring` change — four visible steps, in that order."* | The gate, and its ordering |
| `ADR-001` | *"**Answer key first, before any model.** For each commit, hand-write the findings an excellent senior reviewer should raise"* | What counts as an experiment |
| `REVIEW-freeze-05.md` / `ADR-A009` | A proposed seventh premise was **rejected** because *"it would enter the catalogue with no experiment behind it, which this ADR's gate forbids"* | The on-point precedent: a well-argued addition refused for want of an experiment |

### What the frozen corpus says about inheritance

Searched across every file of the research repository and every pre-existing file of this
architecture repository. The result is more useful than plain silence, and it points the same way.

**`extends`, `parent class` and `@mixin` appear nowhere.** Not once, in either corpus. The only hit
for *inherit* is `README.md`'s heading *"Governing rules (inherited, not invented)"*, which is about
the provenance of principles, not about classes. **No document describes resolving a member through
a class hierarchy, and no answer key names such a member.**

**`trait` does appear — and every occurrence treats a trait as a *diff signal*, never as a
resolution path.** This is the closest the evidence comes to the question, so it is worth being
exact:

| Where | What it says | What it is about |
|---|---|---|
| `experiment-03.md:56` | *"**SoftDeletes removal vs. existing trashed data.** Dropping the trait makes any pre-existing soft-deleted rows visible again"* | a **removed** line in the diff |
| `context-types.md:44` | *"Removing a trait like `SoftDeletes` also changes how existing rows behave"* | the same finding, generalised |
| [ADR-A009](ADR-A009-premise-catalogue.md) trigger table | `data-state-after-behaviour-change` fires when *"a **removed** diff line is a `use <Trait>;` statement inside a class body"* | a literal trigger read from the diff |
| `02-project-structure.md:92` | *"No abstract base classes, no traits shared across groups"* | a rule about **this tool's own** code, not the repository under review |

Experiment 3 is therefore the one place the research met an inheritance-shaped fact — and it
resolved it **without opening the trait's file at all**. The premise is earned from a removed line in
the changed file, at depth *zero*, and ADR-A009 records that every trigger *"is read from the changed
region plus the changed file's own text — the two things R1 already loads. No new input, no new
module, no extra pass."*

That is not silence; it is a worked precedent, and it runs the same way this ADR does. The corpus's
only encounter with inheritance was handled inside the existing depth bound, deliberately, and the
architecture wrote down that this was a property worth preserving.

### Why M1 is not the missing experiment

M1's answer key states what the **tool** should output. ADR-001 defines an experiment as a real
commit plus a hand-written key of the findings **a senior reviewer should raise**, written before
any model output. M1 is a fixture and a regression net — valuable, and cited throughout this ADR for
its measurements — but it is not an ADR-001 experiment, and it cannot discharge ADR-A003's gate. No
answer key in the corpus names an inherited member as required minimum context.

### The motive, honestly stated

A motive for D2 *has* now appeared, and it should be recorded as accurately as its limits:

- **Measured:** 45 of 45 `named_reference` flags on fifteen real commits were false positives (M0
  §4.3); in the M1 fixture, 10 of variant A's 12 flags are false positives, and 4 of those 10 are
  reachable only past D2.
- **Not measured:** whether any of it degrades a review. The research's own open question — *"Is the
  false-positive rate tolerable? Never scored"* (`evidence-gaps.md` §3.2) — is still open, and the
  A/B/C runs that would settle it have never been executed (ADR-001, Consequences).

A count of false positives is a motive to *investigate*. It is not an answer key naming an
ancestor's member as minimum context, and this ADR does not treat it as one.

## Trigger to revisit

Recorded in `evidence-gaps.md` §4 alongside the other under-builds. **D2 is relaxed when, and only
when, an ADR-001-grade experiment exists whose hand-written answer key names a member the changed
file's directly-referenced class does not declare** — that is, a finding a reviewer needed, which
only an ancestry read could have supplied.

Two lesser triggers would each warrant reopening this ADR without relaxing D2 on their own:

- a scored A/C run in which false `unresolved-reference` flags measurably degrade the review; or
- OQ6 being resolved such that the flag's fixed statement becomes *true* for this case — see below.

## The alternative this ADR does not close

**OQ6 — splitting `unresolved-reference` by cause — would fix S05's *correctness* without touching
depth.** The current statement, *"named reference could not be resolved on disk; contract
unverified"*, is false for `Package::query`. A statement scoped to what the run actually did —
*"the member is not declared in the named class's own file"* — would be exactly true at depth one.

That is a cheaper and better-evidenced remedy for the falsity than a traversal is, and this ADR
deliberately leaves it available. Note what it does *not* do: a true-but-uninformative flag is still
noise, so OQ6 addresses ADR-A009's truthfulness requirement, not M0's precision motive. The two
questions are coupled and should be sequenced deliberately.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Allow `extends` walking, bounded to one extra hop** | The bound would be invented, exactly as `--max-call-sites`'s value was (AA1) — but unlike AA1 it would expand a capability rather than bound one. Real chains are not length one (`User extends Authenticatable extends Model`), so a bound of one would resolve some models and not others, producing a rule whose behaviour depends on an application's class hierarchy. Worse than a clean refusal |
| **Allow `extends` walking to a fixed point (whole ancestry), verification only** | The honest form of the proposal, and the one to revisit first if the trigger fires. Rejected now solely for want of an experiment — ADR-A003's gate, freeze review 05's precedent |
| **Treat "one file beyond" as "one *class* beyond", so the class's full inherited surface counts as one hop** | A definitional move that grants the capability by renaming it. `05-traceability.md` and freeze review 01 count *files*, and the tool's own port is `SourceRepository.text(path)`. Redefining the unit to reach a desired answer is the failure mode ADR-A003 exists to prevent |
| **Allow it only for framework classes (under `vendor/`)** | Makes a depth rule depend on which package a file belongs to, which is neither a P3 concept nor a P9 input. It would also be the first Laravel-shaped rule inside the core, which M0 §7.1 rules out |
| **Declare S05's flag correct-by-policy and close the question** | Overstates the evidence in the other direction. The flag's *statement* is demonstrably false; calling it correct would defend a wrong sentence with a policy. This ADR blocks the traversal and records the falsity as open (OQ6) rather than denying it |
| **Defer OQ1 entirely and report only a gap** | The silence is real, but the repository's discipline supplies a determinate answer to silence — under-build plus trigger — and the downstream milestone needs a boundary to build against. Reporting "unknown" would leave the next milestone to decide it implicitly, in code, which is the outcome ADRs exist to prevent |

## Consequences

**Positive**

- The next milestone has an unambiguous, checkable admissibility test — **single-file recognition** —
  and it is stated so that a rule can be judged by reading it.
- The two guarantees D1 and D2 are separated, so a future proposal can be argued against the right
  one. D1 is settled forever; D2 has a named trigger.
- Framework knowledge remains possible without any traversal for two of the M1 fixture's cases —
  facade `@method static` tags and the `scope<Name>` convention — because both are single-file facts.
- The architecture's claim that depth > 1 is structurally unreachable survives intact.

**Negative, and stated plainly**

- Four M1 rows — S02.1, S03.1, S03.2, S05.1 — **cannot be fixed** under this ADR. They will keep
  emitting a flag whose sentence is false. That is a known, recorded defect, not an oversight.
- The Eloquent half of M0's proposed rule **L1** is blocked in its entirety, since it needs to
  establish that a class descends from `Model`. M0's rule **L0** (inheritance walking) is blocked by
  definition.
- Roughly 4 of variant A's 10 false positives, and the largest single group in M0's survey, stay
  where they are until an experiment exists.

**Neutral**

- No module, port, class, interface, premise, assertion kind, lever, or schema field changes. The
  implementation at `v0.1.0` already conforms: `NamedReferenceResolver` opens exactly one file and
  its docblock already states *"it never reads that file's `use` block"*. This ADR does not change
  behaviour; it records why the behaviour is what it is, and what would change it.

## Related

- `evidence-gaps.md` §4 — the under-build and its trigger; §5 — AA12.
- [ADR-A003](ADR-A003-closed-move-set.md) — the four-step gate this decision applies.
- [ADR-A005](ADR-A005-slices-not-files.md) — why an ancestor's source may not enter the bundle.
- [ADR-A009](ADR-A009-premise-catalogue.md) — the truthfulness requirement that makes S05's flag a
  defect, and the OQ6 remedy this ADR leaves open.
- [REVIEW-freeze-05.md](../REVIEW-freeze-05.md) — the precedent: an addition refused for want of an
  experiment.
- M0 research and the M1 fixture, in the implementation repository:
  `docs/research/M0-laravel-framework-knowledge.md`,
  `tests/Acceptance/fixtures/laravel-m1/README.md`.

---

## Addendum (2026-09-24) — the D2 trigger fired; D2 is relaxed for verification only

The trigger above — *"an ADR-001-grade experiment exists whose hand-written answer key names a
member the changed file's directly-referenced class does not declare"* — was met, and recorded
before anything was built: experiment-05's E5.4 (`App\Traits\ApiResponse::success`, `FETCH`,
missed in both vendor modes — M30), the ee5a2e6 held-out key's H.3 (written from the diff alone,
locked, then missed — M31), and the reviewer pre-check in [ADR-A027](ADR-A027-inherited-member-silence-gate-step.md)
§6 (seven of forty-five diff-only cells named the Controller → ApiResponse path).

**What is relaxed.** D2 — *resolution opens at most one file beyond the changed file* — is relaxed
for **verification only**, on exactly the terms §1 and §4 set:

- `AncestryResolver` walks `extends` and `use <Trait>` through **project files** to a fixed point
  and returns a *citation*: declaring type, path, line. It never returns a slice.
- **§4 holds.** No ancestor's source is fetched into the bundle, on any evidence. The one item S1
  produces is a flag whose sentence names where the member is declared (ADR-A028 §5).
- **D1 holds, by construction.** The walk holds a visited-set of resolved type names, constructs
  no `Assertion`, and depends on no extractor. `Oq1DepthBoundaryTest::testD1HoldsStructurally`
  still forbids a worklist everywhere in resolution and the pipeline; the new
  `testTheD2WalkHoldsTypeNamesNotReferences` bounds what the walk may hold and read.
- **The bound is "to a fixed point", not a hop count.** This ADR rejected an invented bound as
  AA1-shaped; ADR-A028 §6 names the fail-closed conditions (a dependency, an unplaceable or
  unreadable type, two declaring traits, a conflict block, a cycle) and each has a test.
- `vendor/` is never opened on this path. A dependency ancestor is named by FQCN from the
  `extends` / `use` clause alone and the walk stops (S2).

**What is not relaxed.** `@mixin` and every other annotation stay unread (§4). `NamedReferenceResolver`
itself still reads only the path the locator placed — `testTheResolverReadsOnlyTheFileTheLocatorPlaced`
is unchanged — and the walk is reachable only from a fourth-form assertion whose placed path is
its own origin file.

`evidence-gaps.md` §4's D2 row is closed by this addendum for the verify-only read; the fetch read
remains an under-build with no motive.

