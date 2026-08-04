# ADR-A007 · A naive, documented token estimate — not a real tokenizer

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R4, P7, P8, "no optimisation before measurement"

## Why this decision exists

P7 requires a token budget and honest accounting, and R4 requires "a running token count against a
budget." Nothing in the research names a tokenizer, a model, or an accuracy requirement. The
architecture must therefore choose the cheapest mechanism that makes budget enforcement honest,
without smuggling in a model dependency.

## Decision

`Assembly\TokenEstimate` derives an item's token count from its character count using **one
documented characters-per-token ratio, declared as a single named constant**. Every bundle item
carries its `tokens` estimate, the bundle carries `used_tokens` and `budget_tokens`, and the JSON
schema labels the number an **estimate**.

The ratio's *value* is an **architectural assumption** (AA4 in
[evidence-gaps.md](../evidence-gaps.md)), not an evidence-derived figure — the same standard
[ADR-A008](ADR-A008-required-budget.md) applies to `--budget`. The architecture fixes that there is
one constant, in one place, labelled an estimate; it does not present the number as proven. It is
also not a port: a pure function with one implementation gains nothing from an interface, and
"replace it later" is future flexibility the measurement rule forbids (freeze review O2, L1).

## Evidence from Phase 0

- P7 asks for a budget with visible drops; it asks nothing about token-count fidelity.
- The hypothesis's cost criterion is comparative and coarse — "dramatically below B", "well under
  half of B's tokens" — a scale at which a fixed-ratio estimate is adequate.
- X5 / P8: a real tokenizer means either a model-specific library or a network call. Both are
  banned.
- "No optimisation before measurement": nothing has yet measured that the estimate misleads a
  budget decision.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Model-specific tokenizer (BPE) library** | Adds a runtime dependency (ADR-A001) and ties the tool to one model's vocabulary; the bundle is model-agnostic by design. |
| **Remote tokenizer API** | Network in a deterministic, offline tool (P8, P9, X5). |
| **Count bytes or words instead** | Same class of approximation but harder to read against a token budget expressed in tokens. |
| **No accounting at all** | Fails P7 and R4; the budget is the mechanism that demonstrates the cost claim. |

## Consequences

- `used_tokens` is approximate and labelled as such; comparisons within a run (item vs item) are
  consistent because one estimator is used throughout.
- The trigger to replace it is a *measured* discrepancy that changes a budget decision — recorded
  in [evidence-gaps.md](../evidence-gaps.md), not scheduled.
