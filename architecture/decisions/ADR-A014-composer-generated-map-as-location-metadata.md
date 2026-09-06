# ADR-A014 · Composer's generated PSR-4 map is read as location metadata, parsed and never executed

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · the last lever on the false positives ADR-A013 left in a bare application map
- **Implements:** R2, P8, P9, P10, ADR-A003, ADR-A012, ADR-A013
- **Adds:** nothing to the contract. No assertion kind, no premise, no lever, no bundle field, no
  `bundle_version` change, no port, no module. One adapter reads one more file.

## Why this decision exists

A standard Laravel application's `composer.json` declares `App\` and nothing else. Every framework
and package name is therefore unplaceable, and ADR-A012 and ADR-A013 — which settle references by
*ownership* — cannot act, because ownership is a question about a path and there is no path.

The fixture measures this exactly. Variant C is the realistic case: the same application as variant
A, with `laravel/framework` v12.64.0 actually installed.

| | `composer.json` psr-4 | `vendor/` | `Illuminate\Support\Str` locatable? |
|---|---|---|---|
| variant A | `App\` | **absent** | no |
| variant B | `App\` + `Illuminate\` (hand-widened) | present | yes |
| **variant C** | `App\` | **present** | **no** — and the file is right there |

Variant C is the gap: the framework is on disk, Composer knows exactly where it is, and the tool
does not look.

## The decision

> **`<vendor-dir>/composer/autoload_psr4.php` is read as additional PSR-4 *location* metadata,
> parsed as text and never executed. Entries the project declares in its own `composer.json` are
> merged first and keep precedence. Ownership is unchanged: it is decided by the path, exactly as
> ADR-A012 decided it.**

The map answers *where a class is*. It does not answer *whose it is*, and nothing about this change
touches the second question.

## Answers to the questions the milestone posed

**Is this source authoritative enough?** It is the most authoritative source that exists. Composer
generates it from every installed package's own `composer.json`, and it is the very map PHP's
autoloader uses at runtime — the tool would be reading the same table the application does. Every
alternative is worse: a hand-maintained namespace list is exactly the catalogue M0 §4.5 measured as
already wrong between two installed Laravel versions, and scanning vendor source to discover
namespaces would be an index, which P9 forbids.

**Is it available in the intended environment?** Conditionally, and this must be designed for rather
than assumed. `vendor/` is gitignored in a standard Laravel application while `composer.lock` is
tracked (M0 finding F7). So the map is present on a developer machine and in CI after
`composer install`, and absent in a bare checkout. Both are first-class cases here.

**Should it be treated as dependency ownership metadata?** No — as **location** metadata only.
Conflating the two would undo M4 and M5. The generated map also carries the project's *own*
namespaces (`'App\\' => array($baseDir . '/app')`), which resolve outside `vendor/` and are
therefore correctly project-owned by the existing path test. One map, two questions, answered
independently.

**Does reading it violate any constraint?** Reading, no: P9 names *"the local repository filesystem,
and the PSR-4 autoload map"* as inputs; the file is inside `--repo` and is reached through
`SourceRepository`, so root-scoping and symlink refusal protect it like every other read.
**Executing it would.** `require` on a generated PHP file means running code from a repository the
tool does not control, bypassing the root-scoping every other read goes through, and making output
depend on arbitrary code — against P8's determinism and the trust boundary
`01-architecture.md` §2 draws. So the file is **parsed as text**. Its grammar makes that safe rather
than clever: the whole file contains only `array()` and `dirname()` calls and only two variables,
`$vendorDir` and `$baseDir`, so a literal-array reader needs no PHP evaluation at all.

**What happens when `vendor/` is absent?** Nothing is merged and behaviour is exactly what it was:
the class stays unplaceable and the reference stays flagged. No crash, no guess, and — importantly —
no pretending an unresolved reference was resolved. That is variant A, and it stays byte-identical.

**Absent or malformed `autoload_psr4.php`?** The same conservative fallback. The reader returns the
entries it can parse and nothing else; a file it cannot read at all yields zero entries. This
mirrors how the adapter already treats an unreadable or undecodable `composer.json` — it returns an
empty map rather than raising — so the failure mode is uniform and deterministic.

**Multiple dependency prefixes?** All are merged; the fixture's map carries 70. `pathFor()` already
orders prefixes longest-first and falls through when a directory does not hold the file, so no new
resolution logic is needed.

**Overlapping prefixes?** Already correct, and already exercised. `Illuminate\Support\` maps to four
directories and `Illuminate\` to one; `Illuminate\Support\Arr` is found under `Collections/` by the
longer prefix, and `Illuminate\Support\Str` only by falling back to the shorter one. M1's scenario
S06 exists for this and keeps passing.

**Does the project's own namespace always retain precedence?** Yes, by merge order: `composer.json`'s
`autoload` and `autoload-dev` entries are read first, and a generated entry for a prefix the project
already declares is **appended after** the project's directories, never in front of them. A project
that declares a prefix therefore always has its own directory tried first, and only falls through to
a dependency's when it genuinely does not hold the class — which is what Composer itself does.

## The invariant this must not break

> Making a dependency class **locatable** must not make its source **fetchable**.

That is guaranteed by construction rather than by care: ADR-A012 withholds a dependency class's
surface and ADR-A013 withholds a declared dependency member, both keyed on the path, and both run
after location. Widening the map moves a reference from *"unplaceable → flag"* to
*"placed → settled with a citation"*, and never to *"placed → fetched"*. Variant B has demonstrated
that path since M4; variant C now demonstrates it for a realistic application, and the fixture
asserts that no bundle item anywhere carries a `vendor/` provenance.

This is the ordering M4 and M5 were sequenced to establish first. Landing this change before them
would have produced the 13,426-token bundle M1 measured.

## What this deliberately is not

- **No Laravel-specific anything.** No namespace list, no `Illuminate\` anywhere in the source. The
  reader is a Composer reader and treats `Symfony\`, `voku\` and `App\` identically.
- **No index.** One file is read once per run and discarded; nothing is built, cached or maintained.
- **No vendor scanning.** Namespaces come from Composer's own declaration, never from walking
  directories.

## Consequences

**Positive**

- Variant C — the realistic installed application — gains the ownership behaviour M4 and M5 built:
  facades settle with their tag citation, declared dependency members settle with a line citation,
  and a bare dependency class settles with its path. Its false `unresolved-reference` flags go.
- The remaining `missing PSR-4 entry` diagnostics now mean what they say: the class is genuinely not
  installed, rather than merely not declared by the application.
- Variant B's hand-widened `composer.json` stops being the only way to reach this behaviour, so the
  fixture device it was built as is no longer load-bearing.

**Negative, stated plainly**

- The tool's output now depends on whether `composer install` has run. Two checkouts of the same
  commit can produce different bundles. That is honest — the repository genuinely contains different
  files — but it weakens the "same diff, same repository state" reading of P8 to "same diff, same
  repository **contents**". Determinism for a fixed tree is unaffected.
- A bare checkout is unchanged, so a CI job that reviews a PR without installing dependencies gets
  none of this. That is M0's recorded risk **R3** and the remedy is operational, not architectural.

**Neutral**

- `composer.lock` is not read. It would name versions but not paths, and paths are the question.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **`require` the generated file** | Executes PHP from the repository under review, bypasses root-scoping, and makes output depend on arbitrary code. The file's grammar is simple enough that parsing costs nothing in return |
| **Read `vendor/composer/installed.json` instead** | It names packages and versions, not PSR-4 roots; the roots would then have to be inferred |
| **Ship a namespace list for known frameworks** | The catalogue M0 measured as already wrong across two installed Laravel versions, and the milestone excludes it |
| **Walk `vendor/` to discover namespaces** | An index. P9 forbids it, and Composer has already done the work |
| **Merge the generated map *instead of* `composer.json`'s** | Loses the project-precedence guarantee, and would make the tool blind to a project whose dependencies are not installed |
| **Treat generated entries as a separate map consulted only on miss** | Equivalent in effect to appending, but adds a second lookup path for no gain. Appending keeps one ordered map and one resolution rule |

## Related

- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) and
  [ADR-A013](ADR-A013-dependency-members-are-settled-not-fetched.md) — the ownership rules this
  makes reachable, and the reason this change had to come after them.
- [ADR-A011](ADR-A011-framework-known-recognition.md) — the facade rule, now reachable in a
  realistic application.
- `01-architecture.md` §2 — the trust boundary that forbids executing the file.
- The M1 fixture: variants **A**, **B** and the new **C**.
