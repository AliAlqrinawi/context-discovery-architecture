# ADR-A027 · A member reached through an ancestor is silently dropped at extraction — the gate is opened, and nothing changes

- **Status:** Accepted
- **Date:** 2026-09-23
- **Phase:** Phase 6 · the first two steps of [ADR-A003](ADR-A003-closed-move-set.md)'s gate
  (hypothesis and experiment) for the gap E5.4 and H.3 measured. **Not** the third or fourth.
- **Implements:** ADR-A003, P3, P6, P10, ADR-A010, ADR-A015
- **Changes:** **nothing.** No production code, no boundary, no premise, no assertion kind, no
  framework table entry, no schema. This ADR records where the gap is, what it would cost to
  close, and why the evidence in hand does not support closing it.

## Why this decision exists

Two context keys, on two independent commits, called the same reference `FETCH`, and the engine
missed it both times:

| Key | Commit | Row | Reference | Result |
|---|---|---|---|---|
| experiment-05 (in-sample) | `ec92403` D1 | E5.4 | `App\Traits\ApiResponse::success` | missed, both vendor modes (M30) |
| held-out-ee5a2e6 (held-out, key-first) | `ee5a2e6` | H.3 | the same trait, `::success / ::created / ::deleted` | missed, both vendor modes (M31) |

The controllers in both diffs return through `$this->success(...)`, `$this->created(...)` and
`$this->deleted(...)`, inherited from the abstract `App\Http\Controllers\Controller`, which is
six lines - `abstract class Controller { use ApiResponse; }` - and declares nothing. The member
is two files away from the changed one.

[ADR-A015](ADR-A015-depth-boundary-trigger-evaluated-and-not-met.md) evaluated E5.4 as a
trigger for ADR-A010's depth boundary and declined it, noting in §2 that the blocker is
extraction, not depth. H.3 re-satisfies the trigger's letter on a held-out commit. This ADR
re-traces the chain on `v0.2.0`, corrects a population figure ADR-A015 measured on the wrong
form, costs the options, and reads the reviewer record for evidence that is not the key author's.

## 1 · Where the boundary is: extraction, line 85

Traced on `v0.2.0` (`e7c919d`) source, not on ADR-A015's account of it.

| Step | Where | What happens to `$this->success(...)` |
|---|---|---|
| recognised | `OwnFileAssertionExtractor::siblingCallsIn`, `src/Discovery/Extraction/OwnFileAssertionExtractor.php:163-199` | the `$this` `->` `T_STRING` `(` pattern matches; `success` enters the candidate list |
| **dropped** | `OwnFileAssertionExtractor::forRegion`, **line 85**: `if (!in_array($name, $members, true)) { continue; }` | `$members` is `MemberSlicer::memberNames($fileText)` - the changed file's own declarations. `success` is not among them. **No assertion is formed.** The line's own comment: *"not a member of this file, so not a sibling this move can reach"* |
| never reached | `NamedReferenceAssertionExtractor::propertyCalls` (lines 144-176) | handles only `$this->property->method(` on a typed, imported property |
| never reached | `OwnFileAssertionExtractor::isClassNamePosition` (lines 121-147) | the recognised class-name positions are `Name::`, `Name $var`, `new Name`, `instanceof Name`, `: Name`, `?Name`. **`extends Name` is not one**, so the parent `Controller` is never asserted either - confirmed by H.12 being vacuous on ee5a2e6 |

**E5.4 never reaches the depth boundary.** Nothing arrives at `NamedReferenceResolver`,
`LeverPolicy` or ADR-A020's surface fallback. The outcome is **silence** - no item, no flag, no
diagnostic - which P10 rates as the worst of the three outcomes, below a false flag. ADR-A010's D2
is real and deliberate; E5.4 is dropped two stages before it.

## 2 · Nine shapes, two causes

| # | Shape | `v0.2.0` | Cause |
|---|---|---|---|
| 1 | `$this->m(`, `m` declared in the file | `same_file_reference`, fetched | - (works: `storedPath`, `apiKey`) |
| 2 | `$this->m(`, `m` in a trait the class itself uses | **silent** | extraction, line 85 - H.4 `resolveLocale`, MISSED on ee5a2e6 |
| 3 | `$this->m(`, `m` declared on the parent class | **silent** | extraction - X8.10 |
| 4 | `$this->m(`, `m` in a trait the parent uses | **silent** | extraction - **E5.4 / H.3**, X8.6 |
| 5 | `$this->m(`, `m` in a dependency ancestor (`getJson`, `hasMany`, `whenLoaded`) | **silent** | extraction - and this silence is the correct answer under every key written so far |
| 6 | `parent::m(`, `self::m(`, `static::m(` | **silent** | `NOT_CLASS_NAMES` excludes them, line 47 |
| 7 | `$this->prop->m(`, typed and imported | `named_reference` emitted | works when `m` is declared on the class; S05-shaped when inherited (resolution, D2) |
| 8 | `Name::m(`, `m` inherited | emitted, **false flag** (+ ADR-A020 surface if project) | resolution, D2 - S05, X8.3, X8.5 |
| 9 | interface default methods | n/a | PHP has none |

**Cause A - extraction.** "A member called on the current object that the file does not declare"
has no extractor. Shapes 2-6 share it.

**Cause B - resolution depth (D2).** Once an assertion exists, `NamedReferenceResolver` opens the
one file the map places for the named class and stops (ADR-A010 §3). Shape 8 is there today;
shapes 2-4 would join it the moment cause A were fixed. Shape 2 needs one hop, shape 3 one,
shape 4 two - ADR-A015's "one hop fixes nothing" holds for shape 4 and not for shape 2.

**E5.4 is one gap in the reviewer's terms and two in the engine's.** Fixing extraction alone
turns silence into the S05 outcome: a flag whose fixed sentence - *"could not be resolved on
disk"* - is false, and stays false because OQ6 was closed as unresolvable
([ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md)).

## 3 · The population, corrected

ADR-A015 §4 counted **190 static call sites** on project classes across `abouelsid-backend/app`
and found zero project-parent one-hop cases. That was the right count for shape 8. It is not the
population for shapes 2-6.

The `$this->m(` form where `m` is not declared in the calling file, counted at corpus HEAD
`450d91f` from the tree alone:

| | |
|---|---|
| sites | **66**, in **40** files, **16** distinct methods |
| in `App\Traits\ApiResponse` (`success` 18, `deleted` 2, `created` 2, `paginated` 1) | **23** |
| in `App\Http\Resources\Concerns\ResolvesLocale` (`resolveLocale`) | **21** |
| dependency ancestors (`hasMany`, `belongsTo`, `whenLoaded`, `user`) | ~10 |
| counter's own misses (own-file members it did not see) | ~5 |

**44 of the 66 are two project traits** - every controller return and every resource's locale
field. This shape is not rare; it is the application's return envelope.

On the three commits the backend has scored, an extraction change would add, counted from the
diffs' added lines as distinct `(file, method)` pairs:

| Commit | New assertions | The key's `FETCH` intent | On the key's `OMIT` rows |
|---|---:|---|---|
| `ec92403` D1 | **10** | 2 (`success`, `deleted` → E5.4) | **8** - test helpers → E5.12 |
| `ee5a2e6` | **8** | 5 (`success` ×2, `created`, `deleted` → H.3; `resolveLocale` → H.4) | **3** - test helpers → H.19 |
| `407c110` | 0 | - | - |

## 4 · The options, costed

Token figures from here on are **estimates** from slice lengths in the corpus, not measurements.

| | Option | Reads | Depth | P6 | What it yields for E5.4 | Estimated cost on D1 / ee5a2e6 | Gate |
|---|---|---|---|---|---|---|---|
| **A** | Extraction only: emit `named_reference` for shapes 2-6 with the parent or trait FQCN read from the changed file | nothing new; `extends` and `use <Trait>` are single-file facts | within one, by any reading | no table, no inference | a **flag with the false sentence** - silence becomes S05 | +10 / +8 assertions, ~360 tokens of flags, plus ADR-A020's surface of `Tests\TestCase` on the test files (~50-150 tokens each); 11 of 18 on OMIT rows | recognised-forms table (`01-architecture.md` §3.3) - a closed interface |
| **B** | A, plus ancestry *verification* to a fixed point, no fetch: walk `extends` / `use <Trait>` through project files, stop at a dependency, settle with a citation *"declared at app/Traits/ApiResponse.php:9; not fetched"* as ADR-A013 does for dependency members | chain length per assertion: 2 for `success`, 1 for `resolveLocale`, 1-then-stop for `getJson` | **relaxes D2** for the verify-only read ADR-A010 §1 disallowed "for now" pending an experiment | language rule only | a **true** citation, no item; E5.4 / H.3 / H.4 stay MISSED as `FETCH` rows, the sentence is right | **zero tokens**; the 11 test-helper assertions settle against dependency ancestors at zero cost but count as assertion-level FPs on OMIT rows | a fixed statement for "declared in a project ancestor" does not exist - ADR-A009; the forms table |
| **C** | B, then fetch the ancestor's slice - what both keys asked for | as B, then the slice | **depth three** in files opened for shape 4 | language rule only | the `FETCH` rows satisfied | ~120 / ~200 tokens of slices (`success`, `created`, `deleted` ≈ 60-80 each, `resolveLocale` ≈ 30); test helpers settle in vendor at zero | **[ADR-A010 §4](ADR-A010-inheritance-and-annotation-traversal.md) forbids fetching an ancestor's source "regardless of any future evidence". Reopening that section is a change to a closed decision, not a gate step. Option C is therefore not on this gate's table** |
| **D** | Framework table: settle `getJson`, `assertSame`, `hasMany`, `whenLoaded` as framework-known | nothing new | within one | **table entries** | nothing - `ApiResponse` is project code and cannot be tabled | removes the 11 test/model false positives from A or B | ADR-A011 gate |

What the scorer cannot see: option B's zero-token settlements are still assertions matched to
OMIT rows, and assertion-level precision would count them as false positives. The
token-weighted number would not. That is a limitation of the metric, recorded in the backend's
ADR-B002; it is not a reason to prefer an option.

## 5 · The case for leaving it, in full

- **Both keys were written by one author**, and reasoned the same way both times - "the
  controller's whole return contract". A second author might reasonably write `OMIT`:
  `ApiResponse` is sixty lines, unchanged since `b2eebe7`, used by every controller in the
  application, and a reviewer of this codebase learns it once. Fetching it into every controller
  bundle - it appears at 18 sites across the app - is Experiment 2's "just in case" precision
  failure under another name.
- **The keys asked for something ADR-A010 §4 forbids regardless.** A `FETCH` row on an
  ancestor's member is a request the architecture has already declined on its own evidence:
  Experiment 1 asked for "the `PlaidAccount` model", never the model and its base. The reachable
  outcome is option B - a true citation - and neither key's row is satisfied by it.
- **ADR-A015 spent this trigger once**, on E5.4 itself, and declined it because the licensed
  remedy fixed nothing measured. H.3 re-satisfies the letter on a held-out commit; it does not
  add a new *kind* of evidence.
- **The reviewer record, read in full** (§6): 7 of 45 diff-only cells named this path as their
  one missing piece. 38 did not.

**What would move the decision toward a change:** a second key author, working from the diff
alone on a controller commit, writing `FETCH` for the trait; or diff-only reviewers naming the
envelope at a rate that survives a corpus not built around controllers. **Toward leaving it:** a
second author writing `OMIT` with the "learn it once" reasoning; or a key-first run of option B
whose settlements land on `OMIT` rows the other author wrote.

## 6 · The pre-check: reviewers who are not the key author

Every diff-only cell with a committed raw answer in M17-M22 was read in full - **45 cells**
(M17 5, M19 7, M20 7, M21 15, M22 11). M23's 11 diff-only cells have no committed raw answers
and could not be read. Q3 asks each reviewer for *the one piece of context they were missing*.

**Seven of 45 named the `Controller` → `ApiResponse` path.** Verbatim:

| Cell | Commit | Q1 | Q3, as written |
|---|---|---|---|
| M19 K1 | `0a6e7d1` | CANNOT_TELL | `app/Traits/ApiResponse.php::paginated` |
| M19 K3 | `4411454` | CANNOT_TELL | `app/Http/Controllers/Controller.php::error` |
| M20 D1 | `ec92403` | YES | `app/Http/Controllers/Controller.php::deleted` |
| M21 C04 | `4411454` | CANNOT_TELL | `app/Http/Controllers/Controller.php::error` |
| M22 K03 | `4411454` | CANNOT_TELL | `app/Http/Controllers/Controller.php::error` |
| M22 K05 | `0a6e7d1` | CANNOT_TELL | `app/Http/Controllers/Controller.php::paginated` |
| M22 K10 | `b2eebe7` | NO | `app/Http/Controllers/Auth/AuthController.php::class AuthController (its extends clause)` |

Four distinct commits. **`4411454` was named by three different reviewers in three milestones.**
`ec92403` is E5.4's own commit. `b2eebe7` is the commit that introduced the trait, and its
reviewer asked for the `extends` clause. Five of the seven abstained (Q1 CANNOT_TELL) with this
as the stated missing piece.

Six of the seven wrote `Controller.php::<member>`. `Controller.php` declares **zero** functions
at every one of those commits; the member each reviewer wanted is in the trait, one hop further.
The reviewers' mental model - "it is on the parent" - is one file short of the truth, which is
precisely what option B's citation would correct and option A's false flag would not.

The count is a rate on a corpus that over-represents controller commits (M17-M22 selected for
non-empty bundles, and controllers produce them). It is evidence that reviewers who are not the
key author name this path unprompted; it is not a rate that transfers.

## Decision

> **No change.** The evidence does not support one, and the strongest objection - both keys were
> written by one author - cannot be answered by building. It can only be answered by a second
> author writing a key.

The trigger is recorded as **evaluated a second time and declined a second time**, on different
grounds from ADR-A015: not that the remedy fixes nothing, but that the experiment which would
decide between the remedies has not been run.

### The two questions, separated

They have been argued as one and they are not:

1. **Whether to fetch an ancestor's slice** - the `FETCH` rows' request, option C. Closed by
   [ADR-A010 §4](ADR-A010-inheritance-and-annotation-traversal.md), regardless of evidence.
   Reopening it is a decision about that section, taken on its own, not a gate step for this
   gap. This ADR does not reopen it.
2. **Whether silence is acceptable** - the outcome at line 85, options A and B. **Open**, under
   P10, which rates a silent omission below a flag. Seven reviewers asked for what the engine
   said nothing about. This ADR does not close it either; it records that closing it needs the
   experiment in §7, and that option A without OQ6 would replace silence with a false sentence,
   which ADR-A009 forbids as surely as P10 forbids the silence.

## 7 · The experiment that would settle it

**Two keys per commit, two authors, before any run.** Corpus: every abouelsid commit whose diff
adds a `$this->m(` call to a member the file does not declare - at least `ec92403`, `ee5a2e6`,
`2996b89`, `e5e48ce`, `b2eebe7`, `4411454` - plus halaw's controller commits from M23 for a
second codebase. The second author is not the author of E5.4 and H.3. Each key writes a row per
such call with `FETCH` / `FLAG` / `OMIT` and its `why`; disagreements are recorded, never
reconciled.

**Three engine commits on a branch, pinned by SHA**, the two-engines-one-`input_key` path the
backend was built for: `v0.2.0` (silence), option A, option B. Option C is not built. Same diffs,
same trees, `installed` only.

**Scored against both keys, per author**, with the existing scorer: assertion precision, item
precision count- and token-weighted, key recall, and the number this turns on - **how many of
the new assertions land on `OMIT` rows written by the other author.**

**The null result, stated so it can be recognised:** the second author writes `OMIT` for
`ApiResponse` on a majority of rows; or option B's settlements are scored FP on more `OMIT` rows
than they gain TP; or the test-helper population (11 of 18 on the two measured commits)
dominates and option D would be needed first. Any of those says: leave the boundary, and take
the silence question (A plus OQ6) separately from the fetch question.

## Flags - nothing made, all gated

| What would be needed | Gate |
|---|---|
| A fixed statement for "declared in a project ancestor; not fetched" (option B) | ADR-A009 |
| `$this->m(` to an undeclared member, and the `extends Name` position, in the recognised-forms table `01-architecture.md` §3.3 (options A-C) | a closed interface |
| Fetching an ancestor's slice (option C) | ADR-A010 §4 - a closed decision, not a gate step |
| Framework table entries for test and Eloquent helpers (option D) | ADR-A011 |
| Assertion-level precision counting zero-token settlements as FPs | backend ADR-B002, a metric note |

## Related

- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) - the boundary, D1/D2, §4.
- [ADR-A015](ADR-A015-depth-boundary-trigger-evaluated-and-not-met.md) - the first evaluation
  of this trigger; §2 first located the blocker at extraction.
- [ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md) - why option A's flag would be
  false.
- [ADR-A003](ADR-A003-closed-move-set.md) - the gate; this ADR is its first two steps.
- Implementation repository: `docs/research/M30`, `M31`; `tests/Acceptance/fixtures/experiment-08`
  rows X8.6 and X8.10; `experiment-19/20/21/22/observations/raw-observations.json` for §6.
- Backend repository: `docs/keys/held-out-ee5a2e6.json` row H.3; ADR-B002.

---

## Appendix · 2026-09-23 · the pre-check changes the decision

The body above is unaltered. Its "no change" was correct on the evidence it had: at the time it
was written, the evidence for the gap was two keys by one author. §6's pre-check is the evidence
it did not have, and this appendix records what that evidence changes.

### What the pre-check found, corrected

- **7 of 45, not 2 of 37.** The figure given when the gate was first opened was a keyword pass
  over the wrong denominator. Every diff-only cell with a committed raw answer in M17–M22 was
  then read in full: **45** cells (M17 5, M19 7, M20 7, M21 15, M22 11; M23's 11 have no committed
  raw answers). Both the numerator and the denominator of the earlier figure were wrong, and both
  corrections are the author's own.
- **`4411454` was named by three independent reviewers across three milestones** — M19 K3, M21
  C04, M22 K03 — each asking for `app/Http/Controllers/Controller.php::error`.
- **`ec92403` — E5.4's own commit — was named by a diff-only reviewer** (M20 D1, asking for
  `Controller.php::deleted`) **independently of the key that wrote E5.4.** This answers the
  one-author objection in §5 directly: the same reference was asked for, on the same commit, by
  a reviewer who never saw the key and by a key author who never saw the reviewer. The objection
  stood on the possibility that only one person ever wanted this. That possibility is closed.
- **Five of the seven abstained** (Q1 CANNOT_TELL) with this as their stated missing piece.
- **Six of the seven wrote `Controller.php::<member>`**, and `Controller.php` declares nothing at
  any of those commits. The reviewers' model — "it is on the parent" — is one file short of the
  truth. Option B's citation, *"declared at `app/Traits/ApiResponse.php:9`; not fetched"*,
  corrects exactly that error. Option A's flag — *"could not be resolved on disk"* — would leave
  it standing and add a false sentence beside it.

### Revised decision

> **Option B is now the candidate: ancestry verification to a fixed point through project files,
> stopping at a dependency boundary, settled with a citation and no fetch, at zero token cost.**
> **Option C stays closed:** no reviewer asked for the slice. Every one of the seven named a file
> and a member — a location — and none quoted or asked for a body. What they lacked was the
> pointer, which is what B supplies and C would exceed.

This is still the gate's **experiment** step. Nothing is built by this appendix. What option B
needs before it can be built, in order:

1. **A fixed statement for "declared in a project ancestor; not fetched."** The catalogue has no
   premise or diagnostic for a member found by walking `extends` and `use <Trait>` through project
   files. ADR-A009 requires a flag's statement to be exactly true of the case it covers and gates
   every addition on an experiment; that experiment is §7's, and this appendix is its evidence,
   not its result. **ADR-A009 gated.**
2. **The recognised-forms table in `01-architecture.md` §3.3.** `$this->m(` to a member the file
   does not declare is not a recognised form, and `extends Name` is not a recognised class-name
   position. Both must be added for an assertion to exist at all — cause A in §2 — and the table
   is a closed interface. **Its own decision, in the interfaces document, not in this ADR.**

Then §7's experiment, with the second author, on the corpus named there, scored against both
keys. Option B is a candidate, not a decision.

### The caveat, kept

The corpus behind 7/45 over-represents controller commits: M17–M22 selected for non-empty
bundles, and controllers produce them. **7/45 is not a transferable rate.** It is evidence that
reviewers who are not the key author name this path unprompted, on four distinct commits; it is
not evidence of how often they would on a corpus chosen otherwise. §7's experiment adds halaw's
controller commits for that reason, and the null result stated there still applies.
