# ADR-A011 · Framework-known recognition is a successful negative, behind one interface, at one call site

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · implements the two rules [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) left reachable
- **Implements:** R2, P2, P5, P6, P8, P9, P10, X2, ADR-A003, ADR-A005, ADR-A009, ADR-A010
- **Adds:** one interface, one value object, one implementation, one constructor argument, one line in `Cli\Wiring`
- **Does not change:** the five assertion kinds, the seven premises, the two levers, the bundle schema, the five ports

## Why this decision exists

The M0 research measured that every `named_reference` flag the tool produced on real Laravel code
was a false positive, and ADR-A010 then fixed the depth bound at **one file beyond the changed
file**. That left exactly two recognition rules reachable — a facade's `@method static` tag and
Eloquent's `scope<Name>` convention — because both are settled by the single file the
`ClassLocator` already places.

Implementing them raises two questions the frozen architecture had to answer rather than the
implementer: **what does a recognised reference produce**, and **where is framework specificity
allowed to live**.

## Decision

### 1 · A framework-known reference is a *successful negative*, not a new outcome

It produces **no bundle item** and **one diagnostic**, through the mechanism freeze review 05
already created for a caller search that completes with zero results
(`01-architecture.md` §4 stage 5d; `03-interfaces.md` §4).

Nothing was invented to make it fit:

| Case | Slices | `lookupRan()` | Bundle | Diagnostic |
|---|---|---|---|---|
| the class's own file declares the member | the slice | — | fetched item | — |
| a naming convention names another member **of that file** | that member's slice | — | fetched item, project-local | — |
| **the framework declares the member** | none | **true** | **no item** | the citation |
| nothing declares it | none | false | flagged item | `unresolved …` |

Row three is required by two frozen rules acting together, and is the only outcome that satisfies
both:

- **ADR-A009** — *"a premise exists only where an unverified premise exists."* The run asked "is this
  contract defined, and where?" and answered with a file and a line. Nothing is unverified, so
  stating `ASSUMPTION: named reference could not be resolved on disk` would be **false**, which is
  exactly the failure freeze review 06 refused when it declined to reuse that premise for a
  caller-search failure.
- **Experiment 2** — its diff carries roughly a dozen `Log::info` calls and its key demands an
  almost-empty bundle, because *"a naked-diff review that knows Laravel catches these unaided."*
  So the framework's own source must **not** be fetched. M1 measured what happens when it is: one
  framework return type expands into 115 slices and 13,426 tokens.

No item, one diagnostic, is the only outcome left. It needs no new assertion kind, premise, lever or
schema field — and none was added.

### 2 · The scope convention resolves to *project* code, and is therefore fetched

`Package::active()` resolves to `Package::scopeActive()`, **in the same file**. The framework
supplies the naming rule; the application supplies the source. It is an ordinary `fetched`
`named_reference` item with provenance in `app/`, indistinguishable from any other project-local
slice — which is correct, because that is what it is. Experiment 1's precedent that a model's own
members are fetched is preserved exactly.

This is the practical content of the distinction ADR-A010 drew: *framework knowledge* and *framework
code* are different things, and only the second is barred from the bundle.

### 3 · The seam is a plain interface in `Discovery\Framework`, not a sixth port

Freeze review **O1/O2** settled the rule: a pure text transformation with one implementation is a
plain class, not a port — which is why `UnifiedDiffParser` sits in `Discovery\Parsing` and
`TokenEstimate` in `Assembly`. `FrameworkKnowledge` performs no I/O: it is handed text the resolver
has already read. Under O1/O2 it is therefore **not** a port, and `Ports` stays at five, as
`ArchitectureBoundaryTest::testThereAreExactlyFivePorts` requires.

The interface still earns its place, for a reason O2 explicitly distinguishes from speculative
flexibility: it is what keeps every other class in `Discovery` free of framework symbols.
`NamedReferenceResolver` depends on the capability's name and never on a framework's; the
implementation is chosen in `Cli\Wiring`, the single construction point.

**The signature is the depth bound.** Both methods take *one file's text* and nothing else — no
path, no `SourceRepository`, no `ClassLocator`. An implementation cannot open a second file, so
ADR-A010 is enforced by the contract rather than by discipline, and `extends`, `use <Trait>` and
`@mixin` are unreachable from behind this interface by construction.

### 4 · The order of consultation — the project always wins

1. the class's own file, asked directly for the member;
2. only on a miss, the naming convention, over that same text;
3. only then, the framework declaration;
4. otherwise the existing `unresolved-reference` flag.

Step 1 is what protects `App\Services\Registry::create` — a project static factory sharing Eloquent's
most recognisable member name. A rule matching on the *name* would swallow application code; this
one never gets the chance, because the project answered first. And a wrong framework answer at step
3 finds nothing and falls through to step 4, so **the failure mode of the whole mechanism is the
behaviour that preceded it**.

### 5 · Framework knowledge is consulted for `NamedReference` only

Never for `SameFileSymbolAbsence`. Experiment 1's headline finding is a *missing `Log` import* — an
absence, and the primary recall test's most important row. A framework layer that "knows about
`Log`" could plausibly suppress it. It cannot reach that extractor, and a test asserts it for every
assertion kind.

## Scope — what was built, and what deliberately was not

**Built.** Facade `@method static` recognition; the `scope<Ucfirst(member)>` convention. Both read
one file. Both cite `laravel/framework` v12.64.0: `Facade::__callStatic()` at
`Support/Facades/Facade.php:355`, the 31 `@method static` tags on `Support/Facades/Log.php`, and
`Model::hasNamedScope()` at `Database/Eloquent/Model.php:1738`, which is literally
`method_exists($this, 'scope'.ucfirst($scope))`.

**Not built, each blocked by ADR-A010 for needing a second file.** `Model::create` and `::where`
(`__callStatic` → `Eloquent\Builder`), `::orderBy` (`@mixin` → `Query\Builder`), `::query` (plain
`extends`), relation members, and any receiver whose type comes from data flow. Their flags stand,
and remain a recorded defect rather than a fixed one.

**Also not built.** `#[Scope]` attribute scopes — the second form `Model::hasNamedScope()` accepts.
It needs attribute parsing and no fixture demands it. Composer-map widening is untouched, so in a
realistic Laravel application (`repo-a-app-map`) `Illuminate\…` is still unplaceable and the facade
rule cannot act at all; it takes effect only where the class is placeable.

## Consequences

**Positive**

- Three M1 answer-key rows close: the two facades in variant B, and the local scope in both. The
  recorded gap moves from `A:match 10 / B:match 10` to `A:match 11 / B:match 13`.
- No framework source enters any bundle. The variant-B facade scenario goes from 2 flagged items to
  **0 items and 0 tokens**, with two cited diagnostics.
- Every guard M1 built holds unchanged: the model surface is still fetched, `Registry::create` is
  still project-local, `MissingGateway::resolve` and the misspelled `Package::activatte` still flag,
  and scenario S09's comments, docblocks, string literals and `::class` still produce nothing.

**Negative, stated plainly**

- The same reference now behaves differently in the two fixture variants — flagged where the class
  is unplaceable, framework-known where it is not. That is a faithful consequence of not widening
  the map, and it makes the M0 problem-A/problem-B split visible in a single row rather than hiding
  it.
- An application facade (`App\Facades\X extends Facade` with `@method static` tags) is recognised
  too. That is judged correct — the proxy *mechanism* is the framework's and no file settles the
  container binding — but it is a consequence worth naming rather than discovering later.

**Neutral**

- `NamedReferenceResolver` now reads its located file at two call sites instead of one. Both read
  the path the locator returned; ADR-A010's bound is about *which* file, not how many reads. The M2
  lock test was corrected to measure that invariant directly, having previously measured call-site
  count as a proxy for it.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **A sixth port, `Ports\FrameworkKnowledge`** | Freeze review O1/O2's criterion — no I/O, one implementation — puts it outside `Ports`, and `testThereAreExactlyFivePorts` locks the count. Considered and reversed during implementation rather than papered over by editing the count |
| **Laravel logic inside `NamedReferenceResolver`** | Makes a core discovery class unusable without Laravel, and is precisely the scattering M0 §7 was written to prevent |
| **A new lever (`known`) or a sixth `assertion_kind`** | Both are `bundle_version` breaking changes, and neither is needed: stage 5d already expresses "answered, nothing to fetch" |
| **A new premise for framework-known references** | ADR-A009 forbids it — there is no unverified premise to state. This is the same seventh-premise proposal freeze review 05 rejected, in a new costume |
| **Fetching the facade or the framework method as a slice** | Refuted by Experiment 2 before it was written, and quantified by M1 |
| **Recognising a facade by the bare parent name `Facade`** | Would claim any application class extending an unrelated class of that name. The parent is resolved the way PHP resolves it — through the file's `use` statements or its `namespace` — and a test covers the unrelated-`Facade` case |
| **Requiring `extends Model` before applying the scope convention** | Needs the parent's file for any real model (`User extends Authenticatable`), which ADR-A010 forbids — and would resolve some models and not others. The convention is checked on its own terms instead: the member is absent, `scope<Name>` is present, in one file |

## Related

- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) — the bound this stays inside.
- [ADR-A009](ADR-A009-premise-catalogue.md) — why no premise was added.
- [ADR-A005](ADR-A005-slices-not-files.md), [ADR-A003](ADR-A003-closed-move-set.md).
- [REVIEW-freeze-05.md](../REVIEW-freeze-05.md) — the successful-negative outcome this reuses.
- The M1 fixture: `tests/Acceptance/fixtures/laravel-m1/`, scenarios S01 and S07.
