# ADR-A009 · Flags come from a closed premise catalogue, not from inference

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R5, X3, X4, P6, P10

## Why this decision exists

R5 requires a flag path for expensive or unknowable dependencies, and the flag is the mechanism
that keeps the tool out of reverse-graph and config-resolver machinery. But "emit an assumption
statement" could mean anything from a fixed sentence to a model-generated judgement. Without a
stated boundary, the flag path is where judgement (X4) and an LLM (X5) would creep back in.

## Decision

`Discovery\Lever\PremiseCatalogue` is a closed list of premise types, each with **one trigger**, one
fixed statement template, and the experiment that earned it. The trigger is what closes the
recognition contract: a premise is emitted **if and only if** its trigger is present. There is no
inference, no scoring, and no "looks risky" path — an implementer needs no heuristic of their own
(P6, X4).

### Triggers (the recognition contract)

Every trigger is read from the changed region plus the changed file's own text — the two things R1
already loads. No new input, no new module, no extra pass.

| Premise | Trigger — emitted if and only if | Earned by |
|---|---|---|
| `surrounding-transaction` | The changed region contains **two or more persistence-write calls** *and* the enclosing member does not itself open a transaction (`DB::transaction(`, `DB::beginTransaction(`). Both facts come from the own-file text | Exp 1 — reconciliation safe only inside a transaction; the transaction lives in the caller |
| `atomic-lock-store` | The changed region names `Cache::lock(` | Exp 3 — lock correctness depends on the deployed store being atomic |
| `schema-index-support` | The changed region names `lockForUpdate(` or `sharedLock(` | Exp 3 — `findConnectedByInstitution(..., lockForUpdate: true)` under a filtered predicate |
| `data-state-after-behaviour-change` | A **removed** diff line is a `use <Trait>;` statement inside a class body | Exp 3 — dropping `SoftDeletes` makes pre-existing trashed rows visible again |
| `unresolved-reference` | A `NamedReference` assertion resolved to nothing — no PSR-4 entry, unreadable path, or member not found | P10 |
| `call-sites-truncated` | A caller search hit `--max-call-sites` | P10, [ADR-A006](ADR-A006-grep-not-graph.md) |

**A premise exists only where an unverified premise exists.** Every trigger above marks something the
run could not settle: a caller not checked, a store not inspected, a migration not read, a lookup that
failed, a search cut short. A question the run *did* settle — a caller search that completes with zero
call sites, a changed file with no `use` block — leaves nothing unverified, so no premise applies and
none is invented. A seventh premise for that case was proposed during implementation and **rejected**:
it would state an assumption that is not being made, which the implementation spec counts as a
precision failure, and it would enter the catalogue with no experiment behind it, which this ADR's gate
forbids (freeze review 05).

Two parts of the first and fourth triggers are wider than the single finding that earned them and are
recorded as architectural assumptions, not evidence: the token list that counts as a
"persistence-write call" (**AA9**) and the widening of trait removal beyond `SoftDeletes` (**AA10**) —
see [evidence-gaps.md §5](../evidence-gaps.md).

**Precision guards, not aspirations.** Experiment 2 must still produce an almost-empty bundle: its
trace-logging diff names no `Cache::lock`, no lock read, removes no trait, and adds no persistence
write, so no trigger fires. Experiment 4 is the guard for the transaction trigger — its key asks for
no premise, so if the trigger fires there, **the trigger is wrong**, and that goes back to the
research repository rather than being tuned here (ADR-A003).

### Statements

One fixed statement per premise:

| Premise | Statement (fixed) | Earned by |
|---|---|---|
| `surrounding-transaction` | "ASSUMPTION: this code assumes a surrounding transaction; caller not checked" | Exp 1 |
| `atomic-lock-store` | "ASSUMPTION: lock correctness depends on the deployed cache store being atomic" | Exp 3 |
| `data-state-after-behaviour-change` | "ASSUMPTION: behaviour depends on whether pre-existing rows are affected in production; no backfill migration present" | Exp 3 |
| `schema-index-support` | "ASSUMPTION: locked/filtered lookup assumes supporting schema indexes; migration not verified" | Exp 3 |
| `unresolved-reference` | "ASSUMPTION: named reference could not be resolved on disk; contract unverified" | P10 |
| `call-sites-truncated` | "ASSUMPTION: additional call sites exist beyond the search bound; not all verified" | P10 / A006 |

Each flag carries the reason (which assertion it resolves) and provenance: the origin **path** and
line span always, and a **member** only when the failing assertion already names one (e.g.
`unresolved-reference` from a `NamedReference`). `Assertion` carries no enclosing-member name and the
flag path has no slicer, so a member is never derived for a flag — the research asks only that the
assumption be stated and attributed to its file (`fetch-vs-flag.md`), and `member` is optional in the
schema. No flag is generated outside this list; there is no free-text or model-generated assumption
path.

## Evidence from Phase 0

- `fetch-vs-flag.md` gives the first three statements almost verbatim as the correct flag texts for
  Exp 1 and Exp 3.
- X3: config and schema are handled "via the generic **flag** path (R5)" until a second commit
  confirms a resolver — so the schema/config premises must exist as flags, and only as flags.
- P6/X4: the engine judges nothing. A fixed statement states an unverified premise; it does not
  assess risk or severity.
- P10: failures must surface as explicit assumptions, which is why the last two entries exist.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Free-text flags composed per finding** | Non-deterministic in practice and unbounded in scope; makes flag comparison across runs impossible. |
| **LLM-generated assumption text** | X5, absolutely. |
| **Infer arbitrary premises heuristically from the diff** | That is judgement (X4) and untraceable to an experiment; the catalogue is what keeps R5 evidence-bound. |
| **A dedicated config/migration resolver instead of premises** | X3: Exp 3 is n=1. Deferred with a named trigger. |
| **Drop the concern when it cannot be fetched** | P10: silent omission is indistinguishable from "nothing needed", which corrupts the precision measurement. |

## Consequences

- Adding a premise requires an experiment, a catalogue entry, and an edit to this ADR.
- If Experiment 5 raises a premise the catalogue lacks, the bundle will show the gap instead of
  papering over it — the intended failure mode ([ADR-A003](ADR-A003-closed-move-set.md)).
