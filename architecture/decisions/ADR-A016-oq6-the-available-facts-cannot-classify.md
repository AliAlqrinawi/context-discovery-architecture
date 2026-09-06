# ADR-A016 · OQ6 is closed as unresolvable — the facts in hand cannot classify a dynamic member

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · closes open question **OQ6**, raised in M0 and named by [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) and [ADR-A015](ADR-A015-depth-boundary-trigger-evaluated-and-not-met.md) as the cheapest remaining lever
- **Changes:** **nothing.** No production code, no premise, no statement, no lever, no schema.

## The question

M8 closed the depth question and pointed at OQ6 as the cheapest next improvement:

> Can unresolved `named_reference` results be classified more accurately using the evidence already
> available at resolution time — no second file, no new premise, no new field, no inference?

96 references in one real application flag as `unresolved-reference` although a Laravel reviewer
needs nothing for them. If the existing evidence could separate those from genuine failures, it
would be a large precision win at zero cost.

## What is in hand when resolution gives up

Seven facts, and no others:

| # | Fact |
|---|---|
| 1 | whether a PSR-4 prefix covers the class name |
| 2 | whether the file that prefix points at exists |
| 3 | whether that path is project source or a dependency's (ADR-A012) |
| 4 | the full text of that one file |
| 5 | whether the member is declared in it |
| 6 | whether `scope<Name>` is declared in it (ADR-A011) |
| 7 | whether the file declares a `Facade` subclass carrying a matching `@method static` tag (ADR-A011) |

Deliberately absent: anything in a second file — the parent, a trait, `Eloquent\Builder`,
`Query\Builder` (ADR-A010, reaffirmed by ADR-A015).

## The decisive measurement

Fixture `experiment-09`, key written first. Three references on the same Eloquent model, evaluated
against all seven facts:

| Member | f1 prefix | f2 file | f3 project | f5 declared | f6 scope | f7 facade | Correct answer |
|---|---|---|---|---|---|---|---|
| `Package::create` | yes | yes | yes | no | no | no | **OMIT** |
| `Package::activatte` | yes | yes | yes | no | no | no | **FLAG** |
| `Package::totallyUnknownThing` | yes | yes | yes | no | no | no | **FLAG** |

**Three references. One fact-set. Three different correct answers.**

No function of the available facts can produce the correct classification, because the facts do not
vary while the answer does. This is a proof, not a judgement, and it does not depend on which rule
anyone proposes.

## The distinction that is real, and the one that is not

`Package.php` declares `class Package extends Model`. That is genuine, in-file, deterministic
evidence — **that the class is an Eloquent model**.

It is not evidence that `create` exists. Those are different facts, and OQ6 fails precisely at the
point where one is silently substituted for the other. `Illuminate\Database\Eloquent\Model` carries
**zero** `@method` tags and **zero** `@mixin` tags, so unlike a facade there is no declarative
surface to read. Contrast row OQ6.7: `Log.php` declares `class Log extends Facade` *and* carries
`@method static void info(...)` — a positive, machine-readable, member-level fact. Eloquent has no
equivalent, and that asymmetry is the whole of OQ6's answer.

## Decision

**OQ6 is closed as unresolvable on the current evidence. Nothing changes.**

Every unresolved `named_reference` keeps its existing outcome and its existing statement. The
`unresolved-reference` premise, its trigger and its fixed text are untouched.

## Options evaluated

| | Option | Evidence | Precision | Typo-safe | Verdict |
|---|---|---|---|---|---|
| **A** | Keep all unresolved references as FLAG | the fact-identity proof | unchanged | yes | **Chosen** |
| **B** | Classify using existing framework knowledge | already done for facades (f7); Eloquent has no f7 equivalent | — | — | Impossible, not rejected — there is nothing to read |
| **C** | A bounded Laravel convention for dynamic Eloquent members | none | would remove 94 false flags | **NO — accepts `activatte`** | Refuted by the fact-identity table |
| **D** | A new assertion / extraction move | this is G1/G5 territory | — | — | Out of scope, ADR-A003 gated |
| **E** | Change the `unresolved-reference` classification itself | — | — | — | Blocked twice over: splitting needs a **new premise** (excluded by this milestone), and broadening was already rejected by [ADR-A009](ADR-A009-premise-catalogue.md), which refuses *"rewording an evidence-derived statement into something vague enough to cover both, losing which lookup failed"* |
| **F** | Sharpen the **stderr diagnostic**, which is not a premise or a schema field | fact 2 distinguishes *no prefix* from *file absent*, and the current text says `missing PSR-4 entry` for both | truthfulness only | yes | **Not taken here** — see below |

### On option F

Fact 2 *is* a real distinction the tool already makes internally and reports wrongly:
`App\Ghost\Missing::create` produces `missing PSR-4 entry` although the `App\` prefix plainly exists
and it is the **file** that is absent. M1's own answer key recorded this as row **S08.1**.

It is not taken in M9, for three reasons: it is a diagnostic-wording defect rather than a
*classification* rule, `experiment-09`'s key did not key diagnostic text, and M8 has just set the
precedent of not acting on findings that surface during measurement. It is recorded as the next
milestone's cheapest item, to be done key-first.

## Consequences

**Positive**

- A line of work is closed with proof rather than left open. M0's proposed rule **L1** — recognise
  Eloquent statics from the model's own file — is now known to be **unimplementable safely**, not
  merely unimplemented. Nobody needs to try it again.
- The asymmetry between a facade and a model is stated once, in terms of evidence rather than of
  framework trivia: a facade publishes its surface, Eloquent does not.
- The typo guard survives untouched, and the experiment now contains three fact-identical rows that
  will refute any future rule of this shape immediately.

**Negative, stated plainly**

- 94 references in one real application keep a flag whose statement is arguably false of them. That
  is the cost of refusing a rule that would also swallow `Package::activatte`, and it is the right
  trade: ADR-A009 exists so a flag is exactly true, and P10 exists so nothing is silently dropped —
  a wrong acceptance breaks both at once.

## Recorded evidence gaps

| Gap | What it would need |
|---|---|
| 94 Eloquent dynamic flags | Member-level evidence that does not exist in one file. Either ADR-A010's bound changes on new evidence (ADR-A015 declined it), or Laravel publishes a declarative surface for `Model`, or the reference is settled by something other than resolution |
| The `missing PSR-4 entry` wording (option F) | A key-first experiment that keys diagnostic text |
| G1 · model surface never fires on `Model::method()` | ADR-A003 four-step gate — untouched by M9 |
| G5 · `$this->inheritedMember()` is silent | An extraction change — untouched by M9 |

## Related

- [ADR-A009](ADR-A009-premise-catalogue.md) — the truthfulness requirement, and the already-rejected broadening.
- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) · [ADR-A015](ADR-A015-depth-boundary-trigger-evaluated-and-not-met.md) — the depth boundary that keeps fact 5 the last word.
- [ADR-A011](ADR-A011-framework-known-recognition.md) — the facade rule, whose evidence Eloquent lacks.
- `docs/research/M9-oq6-classification.md`; fixture `experiment-09/`.
