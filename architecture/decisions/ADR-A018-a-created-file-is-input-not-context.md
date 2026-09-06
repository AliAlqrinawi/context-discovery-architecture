# ADR-A018 · A created file is input, not context

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · closes **G3** and **D1** from the M7 experiment
- **Implements:** R1, R4, P5, P7, P8, P10, AA11, **ADR-A005**
- **Adds:** one field and one method on a `Domain\Diff` value object, and two guards in the extractor
  facade. **No assertion kind, premise, lever, bundle field, `bundle_version`, extraction move or
  resolution-depth change.**

## Why this decision exists

M7 ran the finished tool on a real pull request and found that **59 % of the bundle was the diff
itself**: 730 of 1231 tokens were slices of files the diff had just created. ADR-A005 forbids exactly
this in as many words:

> The reviewer already receives the diff — condition C is "diff **plus** bundle". Duplicating it
> would double-count tokens and confuse the A-vs-C cost comparison.

M7 also recorded **D1**: six items whose stated reason was *"the region uses php, which the file's
`use` block does not import"*. The PHP open tag was being reported as a missing import.

Both surface only on created files, and both were measured before anything was changed.

## The evidence

### G3 — the own-file move was earned on *modified* files

`context-types.md` type 1 is the changed file's own surroundings, and the move exists because
*"absences (missing imports) and near-neighbour contracts (sibling methods) live here and nowhere
else"*. Experiment 1 earned it with `upsertFromPlaid` — a sibling in a **modified** file, whose body
the diff did not show.

When the diff **creates** the file, every line of it is an added line. Its `use` block, its enclosing
member and its siblings are all in front of the reviewer already, directly above the code that uses
them. The move's premise — that this material is off-diff — is simply false for a created file.

### D1 — the open tag became a class name at the token layer

Measured, not guessed. Both extractors tokenise `'<?php ' . $regionText`. When the region text itself
begins with `<?php` — which happens only when the file is new, because `addedLines` holds `+` lines
only — the literal is re-read as three tokens:

```
T_OPEN_TAG  <?php        (the prefix)
(char)      <
(char)      ?
T_STRING    php          <-- and `?` immediately precedes it
```

`isClassNamePosition()` treats a preceding `?` as the `?Name` nullable-type form, so `php` is
recorded as a class name the `use` block does not import.

### A third defect, found while measuring

The extension filter lived in `UnifiedDiffParser::membersIn()` only, so **non-PHP files reached PHP
extraction**. `config/thing.yml` containing `menu:` / `label: Menu` produced assertions about a class
`Menu`, by the very same return-type rule as D1 — `: Menu` reads as a return type. It is fixed here
because it is the same question (*is this text a PHP region at all?*) in the same place, and because
the milestone keyed it: *"No PHP symbol extraction should occur."*

## Decision

> **A file the diff created yields no own-file assertions. A file that is not PHP is not read as PHP.
> Text that already opens with `<?php` is not given a second opening tag.**

A created file's **external** references are extracted exactly as before — a new file may certainly
pull context, just not its own.

| Situation | Before | After |
|---|---|---|
| created PHP file, own-file move | use block, enclosing member, siblings fetched | nothing; one diagnostic per file |
| created PHP file, external references | extracted | **unchanged** |
| modified PHP file | own-file move applies | **unchanged** |
| non-PHP file | tokenised as PHP | no extraction |
| `<?php` in region text | a class name | not a symbol |

### Where each guard lives, and why

- `Domain\Diff\ChangedFile::$isNew` — set by the parser from `--- /dev/null`, the unified format's own
  statement that the file did not exist. The parser had three signals available (`--- /dev/null`, the
  `@@ -0,0` old count, and `new file mode`) and was **discarding all three**; the old-path line is the
  one place the fact is stated unambiguously, so it is read there.
- `ChangedFile::isPhp()` — asked once, on the file, rather than re-derived by each caller.
- `AssertionExtractor` — both guards sit in the facade, because both are facts about the *file* and
  every extractor would otherwise repeat them. The extractor set is untouched and still closed.
- The token layer — fixed in the three extractors that prepend an opening tag, so the tag is invisible
  to symbol extraction rather than filtered out of the bundle afterwards.

**Nothing is filtered after fetching.** The assertion is never created, so nothing is resolved and
nothing is dropped — the duplication is prevented at the source.

### Why this is a narrowing, not a new move

The move set is unchanged: four extractors, three resolvers. `OwnFileAssertionExtractor` still does
exactly what Experiment 1 earned; it is no longer run over input whose premise it does not hold for.
[ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) established that narrowing a move to what
its experiments earned needs no new experiment — only *adding* one does (ADR-A003).

### Why there is a diagnostic

One line per created PHP file: `new file: <path> — own-file context is in the diff, not fetched`.
P10 forbids silent omission, and AA11 already established the diagnostic stream as the channel for a
distinction the bundle does not carry.

## Measurements

**M7's real pull request**, the experiment that raised G3:

| | M7 | M11 | |
|---|---:|---:|---|
| items | 26 | **14** | −12 |
| fetched | 15 | **3** | −12 |
| flagged | 11 | **11** | unchanged |
| used tokens | 1231 | **536** | **−695, −56 %** |
| `same_file_symbol_absence` (all D1) | 6 | **0** | |
| `same_file_reference` (all duplication) | 6 | **0** | |
| items **added** | — | **0** | |
| vendor source in bundle | 0 | **0** | |

The 318 tokens still fetched are `MenuQrService`'s surface, reached from `routes/web.php`. That file
is *also* in the diff, so it is duplication too — but **cross-file** duplication, a different question
that this ADR deliberately does not answer and that no experiment has yet keyed.

**Budget**, same diff: at 500 the bundle is 13 items / 346 used / 1 dropped (M7: 18 / 497 / 8); at
2000 and 8000 it is 14 / 536 / 0 (M7: 26 / 1231 / 0). Priority bands, drop order and accounting are
unchanged; there is simply less to drop.

**M1 fixture:** all **30** runs byte-identical — every M1 diff modifies an existing file, so nothing
there should move, and nothing did. `S10-missing-import-absence` still produces its item, so
Experiment 1's headline finding is intact on the file shape that earned it.

## Consequences

**Positive**

- The largest single precision defect M7 found is closed, and the bundle on a real PR more than
  halves without losing one flag or one external fetch.
- `same_file_symbol_absence` becomes a usable signal again: every instance of it on M7 was the open
  tag, which made a genuine missing import indistinguishable from noise.
- Non-PHP files stop generating assertions about imaginary classes.

**Negative, stated plainly**

- **A genuine missing import in a created file is no longer reported.** For a modified file the `use`
  block is usually off-hunk, which is why absences hide there and why Experiment 1 needed the move;
  for a created file the block sits in the diff directly above the code. That is the justification,
  and it is a judgement, not a measurement — the M11 key contains no created-file-with-missing-import
  row. Recorded as an evidence gap with a named trigger.
- Cross-file duplication remains: a file fetched as a collaborator that the same diff also changed.
  318 tokens on M7.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Filter the bundle after assembly** | The milestone forbade it and it is worse: the work is done, the budget accounting sees it, and the reason strings would still claim the diff does not show the material |
| **Suppress only `same_file_reference`, keep absences** | Tempting, and it would preserve a missing-import signal in new files. Rejected because the *payload* of an absence item is the `use` block, which is equally duplication — and because the M11 key, written first, asked for both to go |
| **Detect new files from the hunk header `@@ -0,0`** | Works, but states the fact indirectly. `--- /dev/null` is the format saying it outright |
| **Strip `<?php` from region text before tokenising** | Mutates the evidence to suit the reader. Not prepending a second tag leaves the text as it is |
| **Drop non-PHP files in the parser** | They *were* changed, and `Diff` should say so; the unreadable-path diagnostic still applies to them. Only PHP *extraction* is wrong |

## Related

- [ADR-A005](ADR-A005-slices-not-files.md) — the rule this enforces at a second site.
- [ADR-A003](ADR-A003-closed-move-set.md) — the move set, unchanged.
- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) — the precedent for narrowing a move.
- `docs/research/M7-experiment-05.md` (G3, D1) and `M11-new-file-duplication.md`; fixture `experiment-11/`.
