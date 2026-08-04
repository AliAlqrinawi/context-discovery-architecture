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

`Discovery\Lever\PremiseCatalogue` is a closed list of premise types, each with one fixed
statement template and the experiment that earned it:

| Premise | Statement (fixed) | Earned by |
|---|---|---|
| `surrounding-transaction` | "ASSUMPTION: this code assumes a surrounding transaction; caller not checked" | Exp 1 |
| `atomic-lock-store` | "ASSUMPTION: lock correctness depends on the deployed cache store being atomic" | Exp 3 |
| `data-state-after-behaviour-change` | "ASSUMPTION: behaviour depends on whether pre-existing rows are affected in production; no backfill migration present" | Exp 3 |
| `schema-index-support` | "ASSUMPTION: locked/filtered lookup assumes supporting schema indexes; migration not verified" | Exp 3 |
| `unresolved-reference` | "ASSUMPTION: named reference could not be resolved on disk; contract unverified" | P10 |
| `call-sites-truncated` | "ASSUMPTION: additional call sites exist beyond the search bound; not all verified" | P10 / A006 |

Each flag carries the reason (which assertion it resolves) and provenance (path, member). No flag
is generated outside this list; there is no free-text or model-generated assumption path.

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
