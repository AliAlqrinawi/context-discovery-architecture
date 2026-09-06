# ADR-A012 · The bare-class surface move applies to the project's own classes

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · resolves the S03.4 over-fetch left open by [ADR-A011](ADR-A011-framework-known-recognition.md)
- **Implements:** R2, R4, P6, P7, P8, P9, P10, ADR-A003, ADR-A005, ADR-A009, ADR-A010, ADR-A011
- **Adds:** one method on an existing port. No assertion kind, no premise, no lever, no bundle field,
  no `bundle_version` change, no new module, no new port.

## Why this decision exists

A reference that names a class and **no member** asks the resolver for that class's **surface** —
every member, sliced. ADR-A011 settled the `Class::member` case; the bare-class case was left open
and measured, on the M1 fixture, as its worst outcome:

```
public function listing(): Collection          →  117 bundle items
                                                  115 slices from one vendor file
                                                  13,426 tokens
```

One return type. The whole of `Illuminate\Support\Collection`.

This is not a framework problem dressed as a precision problem. It is the **surface move applied
beyond what any experiment earned**, and it would behave identically for any large dependency class
in any framework.

## The decision

> **The bare-class surface move applies to a class the project's own source declares. A class the
> PSR-4 map places inside Composer's dependency directory is an installed dependency's: its surface
> is not fetched, its file is not opened, and the reference is settled — no bundle item, one cited
> diagnostic.**

Concretely, for a `NamedReference` whose subject is a bare `Fqcn`:

| The map says | Outcome | Unchanged from |
|---|---|---|
| no prefix covers the name | flag, `unresolved-reference` | v0.1.0 |
| a prefix covers it, the file is absent | flag, `unresolved-reference` | v0.1.0 |
| placed **outside** the vendor directory | **fetch the surface** | v0.1.0 — Experiment 1's model, Experiment 4's enum |
| placed **inside** the vendor directory | **no item, one diagnostic** | **new** |

The diagnostic is
`dependency class: Illuminate\Support\Collection provided by vendor/…/Collection.php; surface not fetched`.

### The discriminator is ownership, not size

A large project class is still fetched in full; a one-member dependency class still is not. "It was
big" is the plausible-but-wrong rule, and it is wrong for three reasons: it would need an invented
threshold (the failure mode AA1 already carries once), it would change behaviour for project-local
references, and it would answer a question about *cost* when the question is about *evidence*.

### Decided from the path, before the file is opened

`isProjectSource()` takes a path and returns a boolean. The check runs **before** the resolver reads
the file, so a dependency's text is never loaded and cannot reach the bundle even by accident. A
test asserts it with a repository spy that fails if anything asks for a `vendor/` file's contents.

## Why this follows from the frozen evidence

### 1 · The spec forbids the payload

`implementation-spec.md`: an item's payload is *"the minimal source slice (**ideally one method or
class member, not a whole file**)"*. 115 slices is a whole file. This is not an interpretation — it
is the same sentence [ADR-A005](ADR-A005-slices-not-files.md) already quotes to forbid whole-file
payloads from the changed file.

### 2 · The move was earned for project classes, and only those

The closed cross-file forms table earns form 3 — *a class name in a `new`, type, or static-call
position* — from **Experiment 1 (`PlaidAccount` model surface)**, and Experiment 4 adds
`App\Enums\PlaidItemStatus`, *"one small class"*. Both are the application's own.
[ADR-A003](ADR-A003-closed-move-set.md): *"The build takes exactly the proven moves and no others."*
No experiment has ever asked for a dependency's surface.

### 3 · Experiment 1 states the size of a correct answer

*"That is one model and three method bodies. **No repository dump, nothing close to it.**"* A
13,426-token payload from a single return type is a repository dump in miniature.

### 4 · Experiment 2 is the standing precision constraint

The spec's precision criterion: on Experiment 2 *"the correct bundle is **almost empty** — pulling
the model 'just in case' is a precision failure, not caution."* Pulling a dependency's entire
surface because it happened to be placeable is the same failure at 300× the cost.

### 5 · The outcome shape already exists

No premise is added, because ADR-A009's rule forbids one: *"a premise exists only where an
unverified premise exists."* The run asked "whose class is this, and where does it live?" and
answered with a path. Nothing is unverified, so `ASSUMPTION: named reference could not be resolved
on disk` would be **false** — the failure freeze review 06 refused. What remains is freeze review
05's successful negative: **no item, one diagnostic**, exactly as ADR-A011 uses it.

### 6 · The input is already there

P9 names *"the PSR-4 autoload map"* as an input, and `ComposerPsr4ClassLocator` already reads
`composer.json`. Composer installs dependencies under `config.vendor-dir` (default `vendor`), which
the same manifest declares. No new input, no new pass, no index, no network.

## What this is not

- **Not Laravel knowledge.** Nothing here names a framework. It applies identically to
  `Endroid\QrCode\Builder\Builder` and to any other package, which is why it lives on
  `ClassLocator` and **not** behind the `FrameworkKnowledge` seam ADR-A011 created.
- **Not a Composer-map widening.** `ComposerPsr4ClassLocator` still reads `composer.json` only.
  Proposal S1 remains unimplemented, and in a realistic Laravel application `Illuminate\…` is still
  unplaceable and still flags — the M1 fixture's variant A is unchanged by this ADR.
- **Not a change to the member case.** `Str::slug` and `Arr::only` still fetch their single declared
  method (397 tokens on the fixture). That is a `Class::member` reference, a different question, and
  it stays open — see below.

## Consequences

**Positive**

- The measured case collapses from **117 items / 13,426 tokens to 2 items / 40 tokens**, with one
  cited diagnostic. Nothing from `vendor/` reaches the bundle.
- The largest single precision defect M1 recorded is closed. The fixture's gap census moves
  `B:over_fetch 3 → 2`, `B:match 13 → 14`.
- A future map widening (S1) becomes safe for bare classes: placing a dependency no longer costs
  anything, because placement and fetching are now separate decisions.

**Negative, stated plainly**

- **Variant A is unchanged.** In a realistic Laravel application the class is unplaceable, so it
  still produces a false `unresolved-reference` flag. That is a problem-A symptom and only S1 can
  reach it. This ADR fixes the over-fetch, not the false flag.
- **The member case is inconsistent with the class case.** A bare dependency class is withheld; a
  dependency *member* that is really declared is still fetched (S06). The evidence for withholding a
  member is weaker — it is one named member, which is exactly what the spec calls a minimal slice —
  so it is left alone rather than swept along.
- A project that installs a dependency **outside** its declared `vendor-dir`, or vendors code into
  its own source tree, is treated as owning it. That is the correct reading of its own manifest.

**Neutral**

- `Ports\ClassLocator` gains a second method. The port count is unchanged at five; the same map
  answers both questions, which is why they share a port rather than justifying a sixth.

## Architectural assumption

**AA13** — treating *"placed inside Composer's `vendor-dir`"* as *"not this project's source"*.
Composer's own convention, declared in the manifest the tool already reads, and not Laravel-specific.
But the research repository never names it, so it is recorded in
[evidence-gaps.md](../evidence-gaps.md) §5 as an assumption rather than as evidence. If a project
organises its source so that the distinction misfires, the correction is to read the distinction
from `autoload` versus the generated dependency map instead of from the path.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Cap the number of slices in a surface** | The threshold would be invented (AA1's failure mode, but expanding rather than bounding), and it would change behaviour for project-local references, which Experiment 1 requires unchanged. It also answers a cost question when the evidence poses an ownership one |
| **Flag a bare dependency class instead** | Its fixed statement would be false: the class *was* located, and ADR-A009 exists so a flag is exactly true. Also the noise M0 measured — 32 of 45 flags were of this shape |
| **Fetch a bounded "signature only" view of the class** | A new payload kind with no experiment behind it, and Experiment 2 says the reviewer needs none of it |
| **Put the rule behind `FrameworkKnowledge`** | It is not framework knowledge. Doing so would make a Composer fact Laravel-shaped and would leave `Endroid\QrCode\…` unfixed |
| **Widen the Composer map (S1) and then fetch** | Explicitly what M1 measured and warned against; it is how this defect was produced in the first place |
| **Do nothing until an experiment demands it** | The defect is not a missing capability but an existing move applied past its evidence. ADR-A003's gate governs *adding* moves; narrowing one to what its experiments earned needs no new experiment |

## Related

- [ADR-A005](ADR-A005-slices-not-files.md) — the "not a whole file" rule this applies to a second place.
- [ADR-A011](ADR-A011-framework-known-recognition.md) — the member case, and the successful-negative shape reused here.
- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md), [ADR-A003](ADR-A003-closed-move-set.md), [ADR-A009](ADR-A009-premise-catalogue.md).
- The M1 fixture, scenario **S03.4**: `tests/Acceptance/fixtures/laravel-m1/`.
