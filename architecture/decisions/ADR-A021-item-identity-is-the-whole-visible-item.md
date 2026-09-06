# ADR-A021 · Two bundle items are the same item when everything a reviewer can see is the same

- **Status:** Accepted
- **Date:** 2026-09-06
- **Phase:** Phase 1 · closes **G4**, recorded by M15
- **Implements:** ADR-A005, P5, P6, P8, P10
- **Adds:** one private method and one loop in `BundleAssembler`. **No assertion kind, no premise, no
  lever, no bundle field, no `bundle_version` change, no new port, no schema change.**

## The gap

M15 found, while measuring something else, that one member named from two files arrives in the
bundle **twice**: two items, identical in every field, 50 tokens for one fact. It recorded the
defect with a key-first trigger rather than fixing it inside another milestone. M16 pulled it.

> Does one identical slice per `(path, member)` always provide the correct reviewer context, or can
> multiple origins of the same member justify multiple copies?

## The fact the decision rests on

The two levers put **different things** in `provenance`:

| Lever | `provenance` names | So two origins produce |
|---|---|---|
| **Flagged** | the **requesting** site — origin path, changed-region span | **two different items**, pointing at two lines a reviewer must visit |
| **Fetched** | the **declaring** site — the slice's path, member, span | **two identical items** |

Verified on `experiment-15`: the two `OrderCalculator::total` items match byte for byte in every
field including `reason`, while the two `Order::create` flags differ, because their provenance is
their origin.

So the origin of a *fetched* item is **not represented in the bundle at all** — not in the second
copy, and not in the first one either. Keeping the duplicate therefore cannot preserve origin
information; it prints the same unlabelled bytes twice. The question *"does the second copy carry
provenance a reviewer needs?"* is checkable rather than a matter of taste, and for fetched items
the answer is **no**.

## Decision

> **Two bundle items are the same item when every field a reviewer can see is the same:** `lever`,
> `reason`, `assertion_kind`, provenance `path`, `member` and line span, and `payload`. When two
> items are the same item, the bundle carries it **once**.

`tokens` is a function of `payload` and adds nothing to the identity. Nothing else in the item
exists.

**Flags are unaffected by construction.** Their provenance is the origin, so two origins give two
different identities and both survive — which is what M15's row T2 keyed and what M7 has produced
since it was first measured.

## Why that unit, and not a smaller one

Every field in the tuple is forced by a row that goes red without it. This is what "the unit of
identity is derived from the answer key" means in practice.

| Drop from the identity | Row that breaks | What is lost |
|---|---|---|
| provenance `member` | **R4** `Calc::total` vs `Calc::subtotal` | two different declarations become one |
| provenance `path` | **R5** `Calc::total` vs `Formatter::total` | two classes become one |
| `assertion_kind` / `reason` | **R6** `ControllerA::helper` reached as a same-file sibling *and* as a named reference | see below |
| everything but `payload` | **R7** `Alpha::run` vs `Beta::run`, identical bodies | the reviewer is told only one class declares it |
| the line span | **R8** a file's `use` block vs one of its members | two regions of one file become one |

**R6 is the row that decides the shape.** `app/Http/ControllerA.php::helper`, lines 18–21, arrives
**three** times with the same path, member, span and payload — as `same_file_symbol_absence`, as
`same_file_reference`, and as a `named_reference` from another file. Those are not copies: they
answer three different questions, carry three different reasons, and `ItemPriority` puts them in
**different drop bands** — 2 for the same-file kinds, 4 for a named reference. Collapsing them would
silently promote or demote the survivor and change what lives through a budget. Any rule keyed on
the slice's *location* alone — `(path, member, span)` — destroys it.

## The rejected boundaries, measured

`experiment-16`, budget 8000, against a key written first.

| | Identity | items | fetched | flags | tokens | rows broken |
|---|---|---:|---:|---:|---:|---|
| **B0** | none (current) | 15 | 13 | 2 | 328 | — *(2 false positives)* |
| **B1** | `(path, member, span)` | 11 | 9 | 2 | 240 | **R6** |
| **B2** | payload text | 9 | 8 | **1** | 197 | **R6, R7, R10** |
| **B3** | slice, provenances merged | 11 | 9 | 2 | 240 | **R6** |
| **B4** | **the whole visible item** | **13** | **11** | **2** | **278** | **—** |
| **B5** | provenance only | 11 | 9 | 2 | 240 | **R6** |

B4 removes exactly the two items the key marks as false positives and nothing else.

**B2 is the cautionary one.** Deduplicating on payload text collapses a *flag*: both `Order::create`
assumption statements are the same sentence, so the reviewer loses one of the two lines they were
being sent to. Cheapest boundary, worst outcome — which is why the identity is derived from the key
rather than from what is easy to write.

B1, B3 and B5 measure identically because all three collapse on the slice's location, and that is
exactly what R6 forbids.

## Where it belongs

**`BundleAssembler`**, and the architecture already says why.

- **Not resolution.** A resolver is a pure function of one assertion with no cross-assertion view —
  the property that makes depth two structurally unreachable (P3). M14's surface guard went in the
  pipeline for the same reason.
- **Not the budget layer.** `BudgetEnforcer` records every drop as *"below budget priority"*, which
  would be **false** of a redundant copy, and it only acts when over budget — so a duplicate would
  survive at 8000 and vanish at 500. That is two behaviours, not one.
- **Assembly.** It is where resolved assertions *become* items, and where **order is already decided
  once, so the JSON and the Markdown are two renderings of one ordering**. Item identity is an item
  question and belongs beside the only other one.

The collapse runs **before** the sort, so the survivor is the first item resolved and the result is
a function of resolution order alone (P8).

## Is a dropped duplicate a silent omission?

No, and the distinction matters because **P10** is the principle most easily violated here.

P10 forbids dropping a *concern*. Nothing is dropped: the item that remains is byte-identical to the
one removed, so every fact, reason, provenance and byte still reaches the reviewer. `BudgetEnforcer`
records its drops because those items are **gone**; there is nothing analogous to record here, and a
diagnostic saying *"an exact copy of an item that is present was not printed twice"* would be noise.

The assembler is also a pure function with no diagnostic sink, and giving it one to announce a
non-event would be a worse trade than the sentence it emits.

## Consequences

**Positive**

- The last measured false positive in the tool is gone: `experiment-16` goes from 2 to 0.
- The identity is stated once, in one place, and every field in it is defended by a keyed row.
- Flags keep working exactly as M15 keyed them, without a special case — their provenance already
  distinguishes them.

**Negative, stated plainly**

- **On the real Experiment 5 pull request this changes nothing at all**: M7 contains zero
  byte-identical items, so the measured benefit there is 0 items and 0 tokens. The defect is real
  and reproducible, but its cost on the one real input available is not yet demonstrated. A reader
  who thinks that makes it premature has a fair point; the counter is that it is 2 of 15 items on
  the fixture, and that leaving a known-redundant item in place is harder to defend than removing it.
- Assembly now holds a rule about *sameness* as well as *order*. That is one more thing in a class
  whose docblock claims a single job.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| `array_unique()` over serialised items | Reaches B4's answer by accident, states no rule, and would silently change meaning if a field were ever added to the schema |
| Deduplicate in the resolver | No cross-assertion view exists there, by design (P3) |
| Deduplicate in `BudgetEnforcer` | Records a false reason, and only fires when over budget |
| Add the requesting origin to a fetched item's provenance, then keep both copies | A **schema change**, and it does not follow from this experiment: the origin is missing from single items too, so it is a separate question about item *content*. Recorded as a gap |
| Keep duplicates because "more context is safer" | Experiment 2: pulling context that adds nothing is a precision failure. A second unreadable copy adds nothing by construction |

## Related

- `docs/research/M16-duplicate-slice-identity.md` — the experiment, the key, the six boundaries.
- [ADR-A005](ADR-A005-slices-not-files.md) — tokens are spent only where they buy something.
- [ADR-A019](ADR-A019-context-the-diff-already-shows-is-not-fetched.md) — the other rule about not
  printing what the reviewer already holds; it withholds by *visibility*, this one by *identity*.
- [ADR-A020](ADR-A020-the-project-class-surface-fallback.md) — its one-surface-per-class guard is a
  narrower instance of the same instinct, kept where it is because it acts before a slice exists.
- Fixture `experiment-16/`, and `experiment-15/` where G4 was found.
