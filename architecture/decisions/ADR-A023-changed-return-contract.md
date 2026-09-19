# ADR-A023 · A body that changes what *kind* of value it returns is a caller question

- **Status:** Accepted
- **Date:** 2026-09-07
- **Phase:** Phase 1 · closes the gap the M26 reproduction found on a real Laravel pull request
- **Implements:** R3, P3, P6, P8, P10, ADR-A003, ADR-A006, ADR-A011
- **Adds:** a sixth `assertion_kind`, a fifth extractor, one method on `FrameworkKnowledge` and one
  on `MemberSlicer`.
  **No new resolver, no new port, no new premise, no new lever, no new bundle field, no
  `bundle_version` change, no depth change.**

## Why this decision exists

`ChangedSignatureAssertionExtractor` reads the parenthesised parameter span and nothing else. Its own
docblock says so: *"Only the parameter list decides. A body-only change … produces nothing."* That is
correct for what it was built for, and it leaves a hole.

The M26 reproduction is one line of a real repository:

```php
-        return $query->get();
+        return $query->first();
```

inside `BranchRepository::getAll(?bool $showInFooter = null): Collection`. The declared signature does
not move, so no signature assertion is raised. The declared return type is not narrowed either, so
PHP raises nothing at the boundary. Every call site that iterates the result is now wrong, and **not
one of them appears in the diff**. This is precisely the shape R3 exists to catch — *who calls this?*
— arriving through evidence R3's extractor cannot read.

The parser makes the omission structural rather than incidental: for a body-only change it records
**zero** `ChangedMember`s, because a member's declaration survives only in the discarded `@@` header
text. There is no signature pair to compare, so widening `ChangedSignature` would not have reached it.

## Decision

> **When a changed region removes a `return` and adds a `return`, both terminal calls are names a
> closed framework table classifies, the two classes differ, and the added return is a **member's
> own** — placeable inside it and not inside a closure declared within it — emit a
> `ChangedReturnContract` assertion for that member and resolve it through the existing
> `CallerResolver`.**

Implemented as `Discovery\Extraction\ChangedReturnContractAssertionExtractor`.

### Why a separate kind rather than widening `ChangedSignature`

The evidence differs. One reads the parameter list; the other reads a return expression against a
framework table. Collapsing them would put framework knowledge inside the signature extractor and
destroy per-move attribution — and `assertion_kind` exists, per `03-interfaces.md` §2, *"so precision
can be measured **per move** after the scored run"*. A move that cannot be scored separately cannot be
withdrawn separately either.

It is also the direction ADR-A003 requires: a new move is a new kind, declared in the closed
enumeration, not a quiet broadening of an existing one.

### Why it reuses `CallerResolver`

Both kinds ask the one question a bounded grep can answer: *who calls this member?* One asks because
the parameter list moved, the other because the returned shape did. The search is identical — one
grep, one scope prefix, one bound — so it is served by the resolver that already exists.

**The resolver count therefore stays at three.** A new *kind* is not a new *resolution move*, and
`ArchitectureBoundaryTest::testTheMoveSetIsClosed` asserts both numbers explicitly: five extractors,
three resolvers.

### Why resolution stays bounded and non-recursive

Nothing here changes depth. `CallerResolver` remains *"one search, one scope, one bound, no following
of what the call sites themselves call"*. The callers of a caller are **not** fetched: that is P3 and
X2, and this ADR does not reopen either. A consumer two hops downstream of the changed member is
outside the bundle by design, not by omission.

The `--max-call-sites` bound applies unchanged. `DiscoverContext`'s truncation guard was extended to
the new kind so an overflow yields the `caller-search-failed` premise rather than a silently short
list (P10) — the M26 reproduction exercises exactly this, truncating at 20 with one stderr diagnostic.

## A callback's return is not the method's

`outer()` below returns whatever `map()` returns — a Collection, before and after:

```php
public function outer(): Collection
{
    return $this->items->map(function ($branch) {
        return $branch->relations->get();   // → first()
    });
}
```

Reading that inner return as the method's own produced *"the body of outer now returns a single value
or null"* — false — and sent the grep after every caller of a method whose contract never moved. The
same mistake arrives in two shapes, and both are rejected:

| Shape | What nests | How it is rejected |
|---|---|---|
| `map(function ($b) { return … })`, `DB::transaction(…)`, `Cache::remember(…)`, a method of an anonymous class | the **line** sits in a nested scope | `MemberSlicer::memberOwningLine()` returns null for a line inside a function body declared within a member |
| `map(fn ($b) => $b->rel->first())` | the **expression**, on the method's own line | the terminal call is read at the top level of the return expression, so `map` is the finisher, not `first` |

**`memberOwningLine()` is a second question, not a replacement.** `enclosingMemberName()` answers
*which member am I reading?* and is what `OwnFileResolver` and the premise extractor need — a line in
a closure is still inside its member for the purpose of fetching that member as context. The new
method answers *whose statement is this?*. Only the second can say that a `map()` callback's return
is not `outer()`'s, and conflating them would have broken the first two callers.

The rule is positional and needs **no list of framework callbacks**: the first function body met is a
declaration's own, and every `function` keyword before that body closes is nested inside it. Deeper
nesting needs no extra work, because the outermost body's span already covers it. Depth is read from
`token_get_all()`, never from raw text, so a brace inside a string, a comment, a heredoc or a `match`
arm cannot open a scope — the fixture that pins this holds all four (ADR-A004).

Arrow functions are handled by the expression rule rather than the scope rule, because `fn` bodies are
single expressions and cannot contain a `return` statement at all.

## The cardinality table, and what it does *not* claim

`FrameworkKnowledge::returnCardinalityOf()` maps a member name to `many`, `one`, `scalar`, or `null`.
It lives beside the facade and `scope<Name>` rules, per ADR-A011: the framework supplies the naming
fact and the extractor supplies none. Unknown names return `null`, and an unknown name never produces
an assertion — a project's own `->fetchThings()` says nothing about cardinality, and guessing would be
the inference ADR-A003 forbids.

**The table is keyed on the member name and nothing else** — no receiver type, no argument shape,
neither of which is among the available facts ([ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md)).
Three names were removed during review for failing exactly that standard, each checked against
Laravel's own source rather than from memory:

| Removed | Why the name alone cannot answer |
|---|---|
| `find` | Laravel declares the conditional return `($id is Arrayable\|array ? Collection<TModel> : TModel\|null)`, and the body branches to `findMany()` on an array. The class belongs to the *argument* |
| `findOrFail` | The **same** conditional return, and its body delegates to `find($id, $columns)` on its first line. Removed for the same reason as `find`, though later: keeping one without the other was an inconsistency the final review caught, with `findOrFail($ids)` → `get()` claiming a shape change that never happened because an array argument had already returned a Collection |
| `chunk` | `BuildsQueries::chunk($count, callable $callback)` is documented `@return bool`; `Collection::chunk($size)` returns a Collection of chunks |
| `toArray` | `Collection::toArray()` is many rows; `Model::toArray()` is one model's attributes |

Losing the detections these would have carried is the conservative direction: silence, not a wrong
subject.

### The claim the table can honestly make

Not *"every entry is universally safe."* **One** remaining entry is known not to meet the strict
name-alone standard, and is recorded here rather than hidden:

- **`get`** — `Collection::get($key)` is declared `@return TValue`, a single element, and
  `Cache\Repository::get($key)` returns a single value. It stays because it is the entry the move
  exists for (`$query->get()` is the Eloquent finisher in the reproduction), and because firing
  requires *both* sides of a return to be table-known and in different classes, which no observed
  facade usage produces.

`findOrFail` was the second such entry and is now removed, because unlike `get` it had nothing to
weigh against the standard: rarity is a reason not to block a release, never a reason to keep a name
the table cannot classify.

The honest statement is therefore: **each entry names a cardinality that holds for the Eloquent and
Collection finisher position the move reads, and one of them can be defeated by an unusual receiver.**
Widening the table, or inferring the receiver, is out of scope and would need its own ADR.

The removals are pinned by `LaravelFrameworkKnowledgeTest`, which asserts that each refused name
answers null with the reason recorded beside it, so re-adding one is a test failure rather than a
quiet regression.

## The post-image repository assumption

The extractor recovers the changed return's line number by finding the added line's text inside the
span the region already declares. That works because `--repo` is the tree **as the diff leaves it**.

This requirement is now stated in the input contract itself — `03-interfaces.md` §1, the `--repo` row
and the Contract list — and **that is its authority.**

[ADR-A022](ADR-A022-experiments-resolve-at-the-reviewed-commit.md) is **not** the authority, and code
must not cite it as one. A022 records a decision about *how the tool is measured* and states in its
own header that it changes *"the experiment harness only. No production file, no schema, no assertion
kind…"*. Its harness happens to satisfy this contract; it does not define it.

The assumption was always present — `AssertionExtractor` has passed `$source->text($file->path)`
alongside post-image line numbers since the beginning, and `UnverifiablePremiseAssertionExtractor`
reads them the same way — but this move is the first to make it load-bearing, which is what obliges
the contract to be written down.

**For this move, failure is silent in the safe direction.** Pointed at a pre-image tree, the added
line is not found, no member is named, and no assertion is raised. Pinned by
`ChangedReturnContractAssertionExtractorTest::testAnAddedReturnTheSpanDoesNotContainYieldsNothing`.

**Correction (M28).** That sentence was true of this extractor and false of the tool. Every extractor
reads the same file text, and `OwnFileAssertionExtractor` reads its `use` block: on a pre-image tree
an import the diff adds is not there, so a symbol the diff uses is claimed as a
`same_file_symbol_absence` that the post-image would never raise. A pre-image tree can therefore
**inflate** the bundle with a false claim, not only shrink it — and in a measured run that reads as
a false positive charged to the engine. `UnverifiablePremiseAssertionExtractor` and
`OwnFileResolver` scan post-image line numbers as well and can name the wrong member. M28 added a
run-level check in `DiscoverContext`: when none of the non-blank lines the diff adds is present in
any changed file, a `post_image_contract_violated` diagnostic is emitted — a count, never a
threshold, and a diagnostic rather than a flag, since this is an input-contract violation and the
precedent for that class is `unreadable path`. It fires on no committed fixture: every one was
verified post-image line by line before the check was written.

## `bundle_version` stays 1

`03-interfaces.md` §2 states the rule: *"Adding an `assertion_kind` value is additive, and no bundle
has been emitted by a working tool, so v1 was never published to break (freeze review 04)."* Freeze
review 04 set the precedent by adding `same_file_reference` and explicitly holding the version.

One line elsewhere reads the other way: ADR-A011's rejected-alternatives table calls *"a new lever
(`known`) or a sixth `assertion_kind`"* a breaking change. That line rejects a **different** proposal —
a `known` kind for framework recognition, rejected as unnecessary because stage 5d already expressed
it — and its parenthetical is superseded by the normative schema rule above. The kind added here is
additive in the same sense `same_file_reference` was.

The new value is inserted into the fixed item ordering at **position 3**, beside the signature change:
the same question about the same member, so a reader meets them together. Every pre-existing kind
keeps its relative order, so no existing bundle's ordering changes.

## ADR-A003's four-step gate

ADR-A003 requires four visible steps for a new move: *"a new experiment, a requirement entry, an ADR,
and a `Wiring` change — four visible steps, in that order."* Recorded plainly:

| Step | Status |
|---|---|
| **Experiment** | The M26 reproduction on `abouelsid-backend`, reproduced end-to-end and pinned as executable fixtures: `tests/Acceptance/ChangedReturnContractTest.php` and `ChangedReturnContractHunkOffsetTest.php`. **It has no `docs/research/M26-*.md` write-up**, which every prior move has |
| **Requirement entry** | R3 in `05-traceability.md`, extended to record that the reverse-caller lookup now has two triggers. No new requirement number was minted: the machinery, the bound and the scope are R3's |
| **ADR** | This document |
| **`Wiring` change** | One line registering the fifth extractor, plus a hoisted `LaravelFrameworkKnowledge` shared with `NamedReferenceResolver` (it holds only `const`, so sharing is safe) |

**The order was not followed.** The implementation and its `Wiring` line landed before this ADR was
written; the ADR was produced during review, once the gate was noticed to be unsatisfied. That is
recorded here for the same reason A022 recorded a measurement defect rather than quietly fixing it —
a programme whose method is *write it down first* has to show when it did not.

## Limitations accepted on purpose

Each was reproduced against the implementation, not imagined:

| Case | Behaviour | Why accepted |
|---|---|---|
| Two members in one hunk both change to the **same** return text | **No assertion at all.** The added text appears twice in the span, so no occurrence is uniquely placeable | The alternative is picking the first match, which is a coin flip between two members. A wrong subject sends the grep after an unrelated method; silence does not |
| One hunk changes returns in **two** members with different text | One assertion, naming the **first** member only; the second member's change is unattributed | Same rule, applied to the first candidate that resolves. Emitting one assertion per member is a design change, not a bug fix |
| A member whose finisher is a project method | Nothing | Condition 2. Guessing is the inference ADR-A003 forbids |
| A consumer two hops downstream | Not fetched | P3, X2. `Public/BranchController` in the reproduction is reachable only through `GetBranchesAction`, so it is outside the bundle by design |
| A region changing returns into two different classes | Reason text reads e.g. *"returns one or scalar"* — the internal class names | Cosmetic; the subject and the lever are correct. Noted so it is not mistaken for a finding later |
| A method's own `return` on the very line a closure opens — `return $x->map(function () {` | Nothing, because that line is inside the callback's span | Over-rejection by one line, in the safe direction. Splitting the line's two scopes would need column positions the region does not carry |
| A `return` spanning several lines, the finisher on its own line | Nothing, because the added line does not start with `return` | Pre-existing to the scope rule; fails closed |

## Consequences

**Positive**

- The R3 question fires on evidence it previously could not read. On the M26 reproduction the tool
  produces **22 items / 562 tokens** — 21 fetched call sites for `getAll` plus one flagged named
  reference — where before it produced no caller search at all.
- The `--repo` input contract is written down, after being relied on implicitly since the beginning.
- The cardinality table's limits are stated in the open, including the two entries that do not meet
  its own standard.

**Negative, stated plainly**

- The bundle grows on any repository with a widely-called finisher: the reproduction truncates at the
  20-call-site bound and spends a `caller-search-failed` flag to say so. That flag is honest, but it
  is a flag a reviewer must read.
- `get` remains in the table on a pragmatic argument, not on the strict standard that removed four
  other names. A future ADR may have to take that back.
- The move has an ADR and executable fixtures but no research write-up. The evidence is a single
  reproduction, not a corpus, and its precision is unmeasured until a scored run says otherwise.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Widen `ChangedSignature` to any body change** | Asks *who calls this?* of every ordinary refactor. The parser also records no `ChangedMember` for a body-only change, so there is nothing to widen |
| **Infer the receiver's type to disambiguate `find`, `chunk`, `toArray`** | Needs a type resolver the tool does not have and ADR-A016 showed the one-file evidence cannot support. The names were removed instead |
| **Use `ChangedRegion::firstLine` as the changed line** | What the first implementation did, and it is wrong: git opens a hunk with context, so the start sits on the blank line between members or inside the member above. Both modes are pinned by `ChangedReturnContractHunkOffsetTest`, which drives real `git diff` output |
| **Give `ChangedRegion` per-line offsets** | A change to the diff parser's contract for a fact the region's span plus the post-image tree already yields |
| **A new resolver for the new kind** | The question is identical to R3's. A second resolver asking one grep of one scope would duplicate `CallerResolver` and break the closed three |
| **Bump `bundle_version` to 2** | Additive value, no published v1 to break — `03-interfaces.md` §2, freeze review 04 |

## Related

- [ADR-A003](ADR-A003-closed-move-set.md) — the four-step gate this move passes through, out of order.
- [ADR-A011](ADR-A011-framework-known-recognition.md) — where framework naming facts live.
- [ADR-A016](ADR-A016-oq6-the-available-facts-cannot-classify.md) — why the receiver cannot be inferred.
- [ADR-A022](ADR-A022-experiments-resolve-at-the-reviewed-commit.md) — the harness that satisfies the
  post-image contract, and does **not** define it.
- `03-interfaces.md` §1 — the authority for the post-image `--repo` contract; §2 — the kind list,
  ordering and `bundle_version` rule.
