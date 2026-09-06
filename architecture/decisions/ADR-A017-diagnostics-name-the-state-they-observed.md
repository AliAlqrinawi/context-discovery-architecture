# ADR-A017 · A diagnostic names the state that was actually observed

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · corrects a diagnostic that asserted the opposite of what happened
- **Implements:** P8, P10, AA11, ADR-A009 (its truthfulness principle, applied to the diagnostic stream)
- **Changes:** one stderr message, and one method on an internal port to make it decidable.
  **No bundle item, lever, premise, provenance, assertion kind, ordering, token count,
  `bundle_version` or schema field changes.**

## Why this decision exists

When a named reference cannot be settled, the tool is in one of three states:

| | State |
|---|---|
| **A** | no PSR-4 prefix covers the class name |
| **B** | a prefix covers it, but no file is there |
| **C** | the file is there, but the member is not declared in it |

`ClassLocator::pathFor()` returns `null` for **both A and B** — it walks the prefixes, and a prefix
that matches but whose directories do not hold the file simply falls through to the next one. The
pipeline therefore reported A and B with a single message:

```
missing PSR-4 entry: App\Ghost\Missing::create named in app/Consumer.php
```

For state B that is **false**, and not merely vague: `App\` *is* declared in `composer.json`. The
message asserts the opposite of the truth, and it sends a reader to fix a mapping that is already
correct instead of to look for a file that is not there.

M1's answer key recorded it as row **S08.1**; M8 and M9 both re-confirmed it while measuring
something else. `experiment-10` shows three of four such lines were false in a single run.

## Decision

> **A diagnostic names the state that was observed. State B gets its own message; state A keeps the
> one it already had, because that message is true of A.**

| State | Message |
|---|---|
| A · no prefix covers the name | `missing PSR-4 entry: <subject> named in <path>` — **unchanged** |
| B · prefix covers it, file absent | `class file not found: <subject> named in <path>` — **new** |
| C · file present, member absent | `unresolved named_reference: <subject> in <path>` — **unchanged** |

Deciding between A and B needs one fact the map already holds: *does any prefix cover this name?*
`Ports\ClassLocator` gains `hasMappingFor(string $fullyQualifiedClass): bool`, answered from the
prefix set in memory. **No file is opened to answer it** — which is forced by the case itself, since
state B is precisely a name whose file does not exist.

### Why not one message covering both

Collapsing A and B into a single vague line would fix the falsity by discarding attribution. That is
the move [ADR-A009](ADR-A009-premise-catalogue.md) already rejects for premises — *"rewording an
evidence-derived statement into something vague enough to cover both, losing which lookup failed"* —
and the reasoning transfers directly. A was already reported truthfully, so A does not change; only
the wrong message is replaced.

### Why this is not a premise change

The flag, its `unresolved-reference` premise and its fixed statement are **untouched**. ADR-A009
governs what enters the bundle; this governs the stderr stream, which AA11 already established as
the channel for distinctions the bundle does not carry. `experiment-10` asserts the bundle is
byte-identical before and after.

## Scope, and the one thing beyond wording

The milestone was scoped to diagnostic text. One thing goes beyond it and is recorded here rather
than slipped through: **`Ports\ClassLocator` gains a method.**

That is an *internal* interface. `03-interfaces.md` §1 states the five ports and the discovery
collaborators are internal and *"may change without notice, since nothing outside the tool depends
on them"*, and [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) added `isProjectSource()`
by the same route. The **public** surface — the CLI contract and the bundle schema — is unchanged.

## Evidence

`experiment-10`, key written before the run. Four unplaceable references under two project prefixes
plus one unmapped namespace:

| Row | Reference | State | Before | After |
|---|---|---|---|---|
| M10.1 | `App\Ghost\Missing::create` | B | `missing PSR-4 entry` ✗ | `class file not found` ✓ |
| M10.2 | `MissingNamespace\Ghost::create` | A | `missing PSR-4 entry` ✓ | unchanged ✓ |
| M10.4 | `Acme\Lib\Thing::make` | B | `missing PSR-4 entry` ✗ | `class file not found` ✓ |
| M10.5 | `App\Foo\Bar\Deep::go` | B | `missing PSR-4 entry` ✗ | `class file not found` ✓ |
| M10.6 | `App\Models\Package::activatte` | C | `unresolved named_reference` ✓ | unchanged ✓ |
| M10.8 | `Log::info` | framework-known | `framework reference: …` ✓ | unchanged ✓ |

M10.4 and M10.5 matter beyond M10.1: they prove the rule reads the declared prefix set rather than
assuming `App\`, and that a deeply nested name is not mistaken for an unmapped one.

Across the M1 fixture, exactly **three** baseline entries changed — all `S08`, all stderr-only, zero
non-stderr fields. On M7's real pull request, **no** diagnostic line changed at all, because that
repository has its dependencies installed and produced no state-B reference.

## Consequences

**Positive**

- A reader is now told which thing to fix: a mapping to add, or a file to find.
- The three states the tool genuinely distinguishes internally are, for the first time, the three
  states it reports.
- `MissingGateway::resolve` — the M1 guard row — now reads `class file not found`, which is what M1's
  key said the truth was.

**Negative, stated plainly**

- Anything parsing stderr for the literal `missing PSR-4 entry` will see fewer such lines. Diagnostics
  are documented as human-facing and explicitly *not* part of the machine contract
  (`03-interfaces.md` §1: the bundle on stdout carries no diagnostics), so nothing supported breaks —
  but the change is real and is named here rather than left to be discovered.

**Neutral**

- One more method on an internal port; ports remain five.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **One vague message for A and B** | Fixes falsity by destroying attribution — ADR-A009's rejected move |
| **Leave it; it is only a diagnostic** | P10's whole point is that what the run could not do is stated *accurately*; a false statement is worse than a vague one, and this one is inverted |
| **Split the `unresolved-reference` premise instead** | A new premise, which ADR-A009 gates behind an experiment, and it would change the bundle. OQ6 (ADR-A016) already closed that route |
| **Report the mapped path that was tried** | More informative and more to go wrong: multi-directory prefixes mean several paths were tried. The smallest truthful message names the state, not the search |

## Related

- [ADR-A009](ADR-A009-premise-catalogue.md) — the truthfulness principle this applies to diagnostics.
- [ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md) — why the *bundle* statement cannot be split.
- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) — the precedent for adding a locator method.
- `docs/research/M10-diagnostic-truthfulness.md`; fixtures `experiment-10/`, `laravel-m1/` row S08.1.
