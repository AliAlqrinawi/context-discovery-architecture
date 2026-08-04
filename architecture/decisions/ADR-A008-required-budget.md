# ADR-A008 · `--budget` is required and has no default

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** P7, R4, "evidence before architecture"

## Why this decision exists

P7 requires "a context budget dramatically below full-repo size", but the research repository never
states an absolute number — the hypothesis speaks only in relative terms, and full-repo (condition
B) token size was never measured. Any default the architecture invented would be a fabricated
value travelling inside a tool built to keep validated and assumed apart.

## Decision

`--budget <int>` is a **required** CLI option with no default. Omitting it is a usage error
(exit `1`) whose message states why: the operator sets the budget for the run being scored.

## Evidence from Phase 0

- `docs/00-introduction/02-hypothesis.md`: the cost condition is "well under half of B's tokens for
  equal quality" — relative to a B measurement that does not exist in the repository.
- `docs/03-phase1/requirements.md` forbids encoding untested assumptions in the implementation; a
  default budget is an untested assumption about acceptable cost.
- P7's purpose is to *demonstrate* cost, which requires the operator to choose the number they are
  demonstrating against.
- `evidence-gaps.md` §2 records B's cost as unmeasured.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Ship a default (e.g. 8 000 tokens)** | Invents a threshold the research does not support and hides it in code, where it would silently shape every later measurement. |
| **Derive the budget from repository size** | That is a heuristic — an optimisation before measurement, and an implicit model of B's cost. |
| **No budget at all** | Violates P7; without a budget the tool cannot show the cost claim, and drops would never be exercised. |
| **Budget as a config file** | Adds a config surface (ADR-A001: no config file) and moves the number out of the run's own command line, where the scored run's parameters should be visible. |

## Consequences

- Every recorded run carries its budget in its command line and in the bundle
  (`budget_tokens`), so a scored table is reproducible.
- The acceptance-test fixtures each declare the budget they were run with, making
  "within budget" a checkable claim rather than an assumption.
