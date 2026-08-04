# ADR-A002 · A pure pipeline behind ports — not events, not plugins, not agents

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** the spec's nine responsibilities, P6, P8, P9

## Why this decision exists

The spec lists nine responsibilities in a fixed order and demands determinism ("given the same diff
and repository state, the engine produces the same bundle"), inspectability, and an acceptance test
that compares a produced bundle to a hand-written key. The internal control flow must be chosen so
that each of those nine steps can be checked in isolation and the whole run reproduced exactly.

## Decision

One synchronous use case, `Pipeline\DiscoverContext`, executing the nine spec responsibilities in
order, passing **values** between stages. Every side effect is behind one of five `Ports`
interfaces, implemented by adapters that are constructed once in `Cli\Wiring`. Pure text
transformations — diff parsing, token estimation — are **not** ports: with no I/O and one
implementation each, an interface would buy nothing (freeze review O1/O2).

No dispatcher, no listeners, no middleware chain, no queue, no worklist, no planner, no retry loop.

## Evidence from Phase 0

- The spec's responsibilities are already an ordered list; the architecture adds no ordering
  freedom the evidence does not ask for.
- P8 (deterministic, inspectable) and P9 (no service to call, no index) rule out anything with
  scheduling, concurrency, or hidden state.
- The acceptance test is a per-commit comparison against a key. Pure stages with value inputs can
  be run with in-memory fakes; an event-driven design would make "which listener contributed this
  item?" a debugging exercise, directly against P5's requirement that every item be traceable.
- P6: the engine judges nothing. There is no decision loop for an agent to run.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Event-driven / observer architecture** | Excluded by the brief; also defeats determinism-by-reading: item provenance would depend on listener registration order. Nothing in the evidence needs decoupled producers. |
| **Plugin system for discovery moves** | Excluded by the brief and by evidence: the move-set is *bounded* (Exp 4 added no new move). A plugin point is an invitation to add unproven moves — precisely what ADR-003 forbids. See [ADR-A003](ADR-A003-closed-move-set.md). |
| **An agent loop that decides what to fetch next** | Requires an LLM in the tool (X5) and iterative re-entry (X2/P3). Banned twice over. |
| **A single god class doing all nine steps** | Passes determinism but fails inspectability: no stage could be tested or scored on its own, and the four extractors could not be measured separately (the `assertion_kind` field exists so precision can be attributed per move). |
| **Framework middleware pipeline** | Drags a dependency (ADR-A001) to express what nine ordered method calls already express. |

## Consequences

- Adding a stage is a visible edit to one file, reviewable against ADR-003's boundary.
- The pipeline cannot "discover" behaviour at run time; what runs is what `Wiring` constructs.
- Concurrency is unavailable by construction — acceptable, since no measurement has shown the tool
  to be slow (no optimisation before measurement).
