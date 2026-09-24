# ADR-A020 · When a project member cannot be resolved, the class's surface is

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · closes **G1**, the gap M7 measured and M13 bounded
- **Implements:** R2, R4, P5, P6, P8, P10, ADR-A005, ADR-A012, ADR-A016
- **Adds:** one method on `NamedReferenceResolver` and one branch in the pipeline. **No extraction
  move, no assertion kind, no premise, no lever, no bundle field, no `bundle_version` change, no new
  port, no depth change.**

## Why this decision exists

M7 measured that Experiment 1's model-surface move — the best-evidenced move in the corpus — **never
fired** on a real Laravel pull request. Real Laravel code writes `Setting::updateOrCreate(...)`; it
does not write `new Setting` or `: Setting`, and only the latter forms reach the surface today.

M13 ran the experiment and found two things that decide the shape of the fix.

### This is a conformance gap, not a new move

The frozen closed-form list (`01-architecture.md` §3.3) row three reads:

> A class name in a `new`, type, **or static-call** position, resolved through the `use` block

`NamedReferenceAssertionExtractor`'s own docblock repeats it word for word — and the code then
excludes exactly that position (`if ($after is T_DOUBLE_COLON) continue; // form 1 already covers
it`). The contract already permits what G1 asks for. **ADR-A003's four-step gate for adding a move
therefore does not apply**; this brings the implementation toward the frozen list, in the same way
[ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) brought it toward `context-types.md` —
that one by narrowing, this one by widening, both needing their own ADR.

### Extraction cannot decide it

`RegionAssertionExtractor::forRegion()` receives `(ChangedFile, ChangedRegion, string $fileText)` and
the extractor holds only a `MemberSlicer`. It has **no `ClassLocator` and no `SourceRepository`**. So
*"is this project code?"* and *"does the member exist?"* are unanswerable there. Restoring the
static-call position in the extractor would give a surface to every `Class::member` — and M13
measured what that costs.

## Decision

> **When a `Fqcn::member` reference cannot be resolved, the class's surface is fetched instead of
> nothing — provided all three hold: the `ClassLocator` places the class, the located path is project
> source, and the member does not resolve in that class's own file. The existing
> `unresolved-reference` flag is retained alongside it.**

This is **M13's boundary B4**, evaluated at resolution time, in the resolution path that already
exists.

### The three conditions, and why each is load-bearing

| Condition | Removing it costs |
|---|---|
| the class is **placeable** | `MissingGateway::resolve` would gain a bare-class assertion and produce a **second flag** for one unknown reference |
| the path is **project source** (ADR-A012) | `Log::info` and every dependency reference would enter the surface path; ADR-A012 exists precisely to keep a dependency's source out of the bundle |
| the member **does not resolve** | `Registry::create` would gain a surface it does not need. Its `create()` is declared, so the minimal slice **is** the member — the spec's *"ideally one method or class member"* and Experiment 4's `PlaidClient::createLinkToken`, *"one method (the mode switch)"* |

### The rejected boundaries, with M13's measurements

| | Boundary | added | slices | tokens | why rejected |
|---|---|---:|---:|---:|---|
| **B1** | frozen form 3, unconditional | 5 | 7 | 135 | **2 false positives** — a surface for `Registry`, and a second flag for `MissingGateway` |
| **B2** | B1 + project source | 3 | 7 | 135 | **1 false positive** — ownership cannot tell a model from an ordinary project class |
| **B3** | member-unresolved only | 4 | 4 | 70 | **1 false positive** — "the member did not resolve" is also true when the class does not exist |
| **B4** | all three | **2** | **4** | **70** | **0 false positives**, 8/8 key rows |

B4 was derived from an answer key written before the measurement, then checked against input it was
not designed from — M7's real pull request, where it adds exactly `Setting`, `MediaItem` and `User`.

## The flag is retained, not replaced

Settled by a second hand-written key (`experiment-14`), and decided by one row.

`Package::activatte` — a typo — satisfies all three conditions **identically** to `Package::create`,
and [ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md) proved that no fact available to
the tool separates them. Under *replace*, the typo would silently become *"here is the model
surface"* with no warning: M0's risk **R1** and the silent omission **P10** forbids.

> A rule that cannot tell a typo from a method must not make a decision that depends on telling them
> apart.

So the flag stays and the surface is added beside it. One assertion yielding both a flagged and a
fetched outcome is not new — the caller-truncation case has done it since freeze review 06.

**The cost, stated rather than hidden:** an Eloquent member reference now produces two items — a flag
whose statement ADR-A016 already records as arguably false, and a surface. That is the price of
keeping a typo visible.

## What is expressly not done

No extraction change. No inheritance traversal, no `Eloquent\Builder` knowledge, no `@mixin`
(ADR-A010 stands untouched — the fallback opens **no** file beyond the one the locator placed). No
Composer-map widening. No framework knowledge moved into extraction. No new port. Nothing is
special-cased on a class name, a namespace, a base class or a method name: `Registry::create` is
excluded by evidence, never by being told that `create` is Eloquent's.

## Expected cost

M13 projected the real M7 pull request moving from **218 tokens to approximately 606** (11 → 18
items), with recall on its answer key going from **1/4 to 3/4** by closing rows **E5.1** (`Setting`
surface) and **E5.2** (`MediaItem` surface). The measured result is recorded in
`docs/research/M14-model-surface-fallback.md`; this ADR states the expectation in advance so the two
can be compared.

## Consequences

**Positive**

- G1 closes. The most-earned move in the corpus fires on the shape real Laravel code actually uses.
- The implementation stops diverging from its own frozen form table.
- Every exclusion is evidence-driven and individually tested: `Registry` by member resolution,
  `MissingGateway` by placement, `Log` by ownership.

**Negative, stated plainly**

- The M7 bundle roughly triples. That is earned context, but it is a real cost and a scored run may
  yet judge some of it unwanted.
- An Eloquent member reference now yields a flag *and* a surface. The flag's statement remains the
  one ADR-A016 could not make true; this ADR does not improve it and does not pretend to.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Restore the static-call position in the extractor | Extraction cannot see the locator or the repository, so it would be B1 — 2 false positives |
| Replace the flag with the surface | Silences `Package::activatte`. See above |
| Fetch the surface only for classes extending `Model` | Requires reading the parent — ADR-A010 forbids it — and ADR-A016 showed the one-file evidence cannot support it |
| Emit a bare-class assertion from the pipeline | Would have the pipeline create assertions outside extraction, eroding the D1 guarantee ADR-A010 marks inviolable |

## Related

- `docs/research/M13-model-surface-extraction.md` — the experiment and the four boundaries.
- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) — the ownership condition.
- [ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md) — why the typo cannot be told apart.
- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) — the one-file bound this stays inside.
- Fixtures `experiment-13/` and `experiment-14/`.

---

## Addendum · 2026-09-24 · a fourth condition: the placed path is not the assertion's origin path

**Source:** [ADR-A029](ADR-A029-recognised-forms-gate-step.md) §6 and the investigation that
followed it. The text above is unaltered; the three conditions stand exactly as written.

> **The surface is fetched only if, in addition, the path the `ClassLocator` places for the class
> is not the file the assertion originated in.** The same condition is mirrored on the bare-class
> route (`resolve()` → `surface()`), where it closes the one way it could otherwise be reached
> today — a file that imports its own fully-qualified name.

**Why.** ADR-A029's fourth recognised form would produce assertions whose subject is the
*calling* class — which is the changed file. `surface()` slices every member the file declares.
For a **created** file every slice is already filtered by `Diff::showsEntirely` (ADR-A019); for
a **modified** file the fallback would fetch every *unchanged* sibling of the calling class,
which the region never called and no reviewer asked for. The own-file move already fetches the
called siblings as `same_file_reference`; the origin file is never a collaborator, and this
fallback exists for collaborators.

**Content-neutral today.** The condition can fire only when a placed path equals an origin path.
Checked by running, not by argument: **0 of 189 `named_reference` assertions** across the eight
succeeded backend runs (D1 ×4, ee5a2e6 ×2, 407c110 ×2; seven of them carry named references),
the M26 golden's two assertions, and all ten `laravel-m1` scenarios. Every recorded bundle is
byte-identical under the condition. **Its test is therefore synthetic** — a class that imports
its own FQCN and calls `Self::undeclared(` — and the test was shown to fail with the condition
reverted.

**Shape.** Path identity, `$path === $assertion->originPath`, asked in full inside
`unresolvedMemberSurface` per this ADR's own rule (*"asked in full here rather than assumed from
the caller, so the boundary holds wherever this is called from"*). Not a class-name comparison:
the harm is dumping a *file's* members, path is the unit ADR-A019 already reasons in, and
**"no name is special-cased" holds** — nothing is excluded by what it is called.

**What it does not do.** It does not touch M13's boundaries B1–B4: `Setting` is never the file
that references `Setting::updateOrCreate`, so E5.1 and E5.2 fetch exactly as before. It does not
bump `POLICY`: `LeverPolicy`, `ItemPriority` and `PremiseCatalogue` are unchanged. It lands
alone, before ADR-A029's form, so that the form's tests inherit the invariant instead of
discovering it; ADR-A029's three-changes-as-one is thereby two.
