# Context Discovery · Phase 1 Architecture Repository

Implementation-ready architecture for the **Phase 1 tool** specified in
[AliAlqrinawi/context-discovery-research](https://github.com/AliAlqrinawi/context-discovery-research)
(`docs/03-phase1/`). This repository contains **no implementation code** by design — it contains
the boundary, the modules, the interfaces, the structure, the diagrams, and the decisions.

Source of truth: the research repository. Every decision here cites a file in it. Where the
research is silent, this repository says so instead of inventing a value (see
[evidence-gaps.md](evidence-gaps.md)).

## What is being built

One deterministic command — **diff in, context bundle out** — that performs only the discovery
moves Phase 0 proved, attaches a *reason* and a *lever* to every item, and enforces a token
budget with a visible drop list. It does not review, judge, score, or call a model.

## Read in this order

| # | Document | What it settles |
|---|----------|-----------------|
| 0 | [00-research-verification.md](00-research-verification.md) | Project, hypothesis, Phase 1 scope, out-of-scope — verified against the repo |
| 1 | [01-architecture.md](01-architecture.md) | High-level architecture, boundaries, modules, responsibilities, data flow, I/O |
| 2 | [02-project-structure.md](02-project-structure.md) | Directory layout, module organisation, domain boundaries, dependency direction, naming |
| 3 | [03-interfaces.md](03-interfaces.md) | Public interfaces (CLI contract, bundle schema) and internal interfaces (port signatures) |
| 4 | [04-diagrams.md](04-diagrams.md) | High-level, request flow, Context Discovery pipeline, module dependency |
| 5 | [05-traceability.md](05-traceability.md) | Every module → requirement → experiment. Nothing untraceable. |
| 6 | [06-acceptance.md](06-acceptance.md) | The acceptance test: bundles reproduced against the four Phase 0 keys |
| 7 | [decisions/](decisions/) | ADR-A001…A009 — the architecture decisions, with evidence and rejected alternatives |
| 8 | [evidence-gaps.md](evidence-gaps.md) | What the research repository does *not* settle, the architecture's response, and the eight recorded architectural assumptions |
| 9 | [REVIEW-freeze-01.md](REVIEW-freeze-01.md) · [REVIEW-freeze-02.md](REVIEW-freeze-02.md) · [REVIEW-freeze-03.md](REVIEW-freeze-03.md) | The freeze reviews: findings, corrections applied, the freeze verdict, and the two implementation blockers closed |

## Status

**Implementation ready** at [REVIEW-freeze-03.md](REVIEW-freeze-03.md). Frozen at
[REVIEW-freeze-02.md](REVIEW-freeze-02.md) — 5 ports (was 7), 25 classes, one CLI option and one
schema field removed, fourteen corrections applied, eight architectural assumptions recorded. Freeze
review 03 then closed the two implementation blockers: the premise recognition contract (one literal
trigger per premise) and the acceptance/implementation inconsistency on Experiment 1's caller
(`fetch-expected` vs `flag-satisfied`). No module, capability, or scope changed.

## Governing rules (inherited, not invented)

Taken verbatim in spirit from `docs/03-phase1/architecture-principles.md` (P1–P10) and
`docs/04-decisions/ADR-003-phase1-scope.md`:

- Evidence before architecture. No module exists without a requirement and an experiment.
- Assertion-resolution, not file-finding (P1).
- Two levers, cost-governed: fetch cheap/named/depth-one, flag expensive/unknowable (P2).
- Depth one. No transitive traversal (P3).
- No forward import-following — banned at the architecture level (P4).
- Every item self-justifying: reason + lever (P5).
- The engine judges nothing (P6).
- Budget-bounded with visible drops (P7).
- Deterministic and inspectable (P8).
- Inputs free and local; no index, no graph, no network (P9).
- Fail toward flagging, never toward silence (P10).

## Non-negotiable exclusions

No SaaS. No AI agents. No event-driven architecture. No plugin system. No multi-language
support (PHP/Laravel target only — the PSR-4 map is an input). No optimisation before
measurement (no caches, no parallelism, no indexes). No implementation code in this repository.
