# ADR-A019 · Context the diff already shows in full is not fetched again

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · closes the cross-file half of the duplication [ADR-A018](ADR-A018-a-created-file-is-input-not-context.md) began
- **Implements:** R2, R4, P5, P7, P8, P10, **ADR-A005**
- **Adds:** one method on `Domain\Diff\Diff` and one filter in the pipeline. **No assertion kind,
  premise, lever, bundle field, `bundle_version`, extraction move or resolution-depth change.**

## Why this decision exists

ADR-A018 stopped a **created** file being fetched back as *its own* context. It left the other half:
a file that some *other* changed file references, which this same diff also changes. On M7's real
pull request that was **318 tokens** — `MenuQrService`'s whole surface, fetched because
`routes/web.php` names it, while the diff creates `MenuQrService.php` three hunks earlier.

ADR-A005 forbids it in the same words it forbids the own-file case:

> The reviewer already receives the diff — condition C is "diff **plus** bundle". Duplicating it
> would double-count tokens.

## The trap this ADR exists to avoid

The obvious rule — *the declaring file appears in the diff, so do not fetch it* — is **unsafe**, and
the experiment was built to prove it before any code was written.

A **modified** file shows only its hunks. `experiment-12`'s `HiddenService.php` is changed by the
diff, and its only hunk is `@@ -26,6 +26,6 @@` while the referenced `run()` is declared at line 7.
A path-level rule answers "already shown" and deletes context the reviewer has never seen. That is a
false negative, which P10 rates as the worst outcome available.

## The evidence model

Five states, kept separate because they are different evidence:

| | State | Shown? |
|---|---|---|
| **A** | modified file, the declaration lies **wholly inside** a changed region | yes |
| **B** | modified file, the declaration lies outside every region | **no** |
| **C** | **created** file — every line is an added line | yes |
| **D** | modified file where only unrelated content changed | **no** (a case of B) |
| **E** | the file does not declare the subject, or is not in the diff at all | **no** |

Containment must be **total**. A declaration that merely overlaps a region is half-shown, and half is
not the contract — `experiment-12` keys overlapping-start, overlapping-end, straddling, and
spread-across-two-hunks as *not shown*.

## Decision

> **A fetched slice is withheld when, and only when, the diff shows that exact span in full** — the
> declaring file was created, or one changed region contains the span entirely. When every slice for
> a reference is withheld, the reference is **settled**: no bundle item, one diagnostic.

`Domain\Diff\Diff::showsEntirely(path, firstLine, lastLine)` answers it. Line numbers on both sides
are post-change — a region's from `@@ … +start,count @@`, a slice's from the checkout under review —
so the comparison is arithmetic on data the parser already holds. No text matching, no inference, no
heuristic, no path list.

### Scoped to cross-file references, deliberately

The filter applies to `NamedReference` slices only. Own-file slices are exempt, and the experiment
shows why: the own-file move exists to supply *what the hunk does not show*, and for a modified file
the enclosing member frequently sits inside its own region. M1's `S10-missing-import-absence` has
`record()` declared at line 7 inside hunk `@@ -4,4 +4,8 @@` — a uniform rule would suppress it and
delete Experiment 1's headline finding. Created files are already handled by ADR-A018.

### Why the pipeline, not the resolver

"Where does this live?" is resolution's question and it is unchanged. "Does the reader already have
it?" is a bundling question, and the pipeline is where the diff and the resolved slices meet. The
resolvers stay diff-agnostic, `AssertionResolver` keeps its signature, and nothing is filtered after
assembly — the slice never becomes an item.

### Why no new outcome was needed

A reference whose every slice is withheld has been **settled**, not failed: the contract was found
and the reader holds it. That is freeze review 05's successful negative — no item, one diagnostic —
the same shape ADR-A011, ADR-A012, ADR-A013 and ADR-A018 all reuse. ADR-A009's rule holds: nothing is
unverified, so no premise applies and none was added.

## Evidence

`experiment-12`, key written first. One changed file referencing six others across five states:

| Row | Reference | State | Expected | Result |
|---|---|---|---|---|
| M12.1 | `NewService::run` — created file | C | do not fetch | ✓ |
| M12.1b | `?NewService` bare surface — **M7's shape** | C | do not fetch | ✓ |
| M12.2 | `VisibleService::run` — line 12, hunk 11-16 | A | do not fetch | ✓ |
| **M12.3** | `HiddenService::run` — line 7, hunk 26-31 | **B** | **FETCH** | ✓ |
| M12.4 | same file, only unrelated content changed | D | **FETCH** | ✓ |
| M12.5 | created file with docblock/comment/string/`::class` noise | C | no items at all | ✓ |
| M12.8 | `Untouched::go` — not in the diff | E | **FETCH** | ✓ |
| M12.9 | deleted file | boundary | record only | recorded |
| M12.X | schema, kinds, premises, levers, priority, accounting | — | unchanged | ✓ |

Fixture totals: **7 items / 194 tokens → 2 items / 43 tokens**, and the two that survive are exactly
M12.3 and M12.8 — the two the key says must never be suppressed.

**M12.9 · deleted files.** Measured, not invented: a deleted file parses with `isNew` false, the old
path, an empty region span (`lastLine < firstLine`) and only removed lines; its text is absent from a
post-change checkout, so `ClassLocator` cannot place it and the reference flags with ADR-A017's
wording. `showsEntirely()` returns false for an empty span, so nothing is suppressed on that path.

## Measurements — the real pull request

| | M7 | M11 | **M12** |
|---|---:|---:|---:|
| items | 26 | 14 | **11** |
| fetched | 15 | 3 | **0** |
| flagged | 11 | 11 | **11** |
| used tokens | 1231 | 536 | **218** |

**−82 % against M7**, with every flag intact and no vendor source at any point. The three items
removed here are `MenuQrService`'s `LOGO_RATIO`, `logoPath` and `png` — 318 tokens — suppressed
because `MenuQrService.php` carries `new file mode 100644`.

Worth stating plainly: this PR's bundle is now **entirely flags**. That is a property of the pull
request — almost every file in it is new, and what it references is either in the diff or is
Eloquent dynamic dispatch (ADR-A016) — not of the rule.

**Budget:** at 500, 2000 and 8000 the bundle is identical (11 items / 218 used / 0 dropped). The
duplicates now disappear *before* budgeting rather than being dropped by it, which is the intended
ordering. Priority bands and drop semantics are untouched.

**M1 fixture:** all **30** runs byte-identical. `S10-missing-import-absence` and `S07`'s `Registry`
surface both survive.

## Consequences

**Positive**

- The duplication ADR-A005 forbids is now closed on both sides, own-file and cross-file.
- The bundle on a real pull request is 82 % smaller than when the tool was first measured end to end,
  without losing one flag or one genuinely external fetch.
- Every suppression is reported, so the omission is visible rather than silent (P10, AA11).

**Negative, stated plainly**

- The tool now depends on the checkout's line numbers agreeing with the diff's post-change numbering.
  That is already the contract — `--repo` is *"a checkout of that codebase at the commit under
  review"* — but M12 is the first decision to rely on it arithmetically. A stale checkout could
  suppress the wrong span. It cannot fetch the *wrong* content, only withhold content it should have
  fetched, so the failure mode is a false negative with a diagnostic naming it.
- Own-file slices in a **modified** file remain unfiltered even when the hunk shows them in full.
  That is deliberate — see above — and is recorded as an open question rather than a defect.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Suppress when the declaring path appears in the diff** | Unsafe, and `experiment-12` M12.3 exists to prove it: a modified file shows only its hunks |
| **Suppress only for created files** | Safe but leaves state A — a declaration a hunk shows in full — duplicated. The key asked for both, and containment is no harder to establish than `isNew` |
| **Match the declaration text against the diff's added lines** | Text matching where arithmetic suffices, and it would break on context lines, whitespace and partial hunks |
| **Filter the assembled bundle** | The work would already be done, the budget would have seen it, and the reason strings would still claim the diff does not show the material |
| **Apply the filter to own-file slices too** | Would suppress M1's `S10` and with it Experiment 1's headline finding. Measured, not assumed |

## Related

- [ADR-A005](ADR-A005-slices-not-files.md) — the rule this closes at its second and last site.
- [ADR-A018](ADR-A018-a-created-file-is-input-not-context.md) — the own-file half, and the `isNew`
  state this builds on.
- [ADR-A009](ADR-A009-premise-catalogue.md) — why a settled reference states no premise.
- `docs/research/M12-cross-file-duplication.md`; fixture `experiment-12/`.
