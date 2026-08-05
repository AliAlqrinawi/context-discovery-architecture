# ADR-A003 · The move set is a closed set of four extractors and three resolvers

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R1–R5, X2, P3, P4, ADR-003

## Why this decision exists

Phase 0's move catalogue is finite and each entry names the experiment that earned it. The
architecture must express that finiteness *structurally*, otherwise the tool becomes the
open-ended context engine the evidence does not justify — and the unrun sloppy commit
(Experiment 5) would be silently absorbed instead of visibly breaking the boundary.

## Decision

Exactly four assertion extractors and three resolvers exist, constructed explicitly in
`Cli\Wiring`:

| Extractor | Assertion kind | Move | Evidence |
|---|---|---|---|
| `OwnFileAssertionExtractor` | `SameFileSymbolAbsence`, `SameFileReference` | read-own-file | Exp 1 |
| `NamedReferenceAssertionExtractor` | `NamedReference` | fetch-collaborator, depth one | Exp 1, 4 |
| `ChangedSignatureAssertionExtractor` | `ChangedSignature` | reverse-caller | Exp 1, 4 |
| `UnverifiablePremiseAssertionExtractor` | `UnverifiablePremise` | return-empty / flag | Exp 1, 2, 3 |

Resolvers: `OwnFileResolver`, `NamedReferenceResolver`, `CallerResolver`. The shared
`RegionAssertionExtractor` / `AssertionResolver` interfaces exist for uniform iteration **only** —
there is no registry, no discovery, no configuration file, and no way to add an implementation
without editing `Wiring` and this ADR.

Dispatch is an explicit `match` on `AssertionKind` in `Pipeline\DiscoverContext`, and every kind has
exactly one destination. That holds because the kinds partition the **moves** one-to-one: at freeze
review 04 the single `NamedReference` kind was split into `SameFileReference` (research context type 1,
read-own-file) and `NamedReference` (type 2, fetch-collaborator), which two extractors and two
resolvers had been sharing. Five kinds, five moves — still four extractors and three resolvers, a
re-partition of existing output rather than a new move, so this ADR's four-step gate is not triggered.
The cross-file forms table in `01-architecture.md` §3.3 consequently holds **three** rows, not four:
the fourth was the same-file sibling, which moved to `SameFileReference`. An earlier draft
gave `AssertionResolver` a `supports()` probe; it was removed at the freeze review because runtime
probing over a closed set hides the set from the reader and is the exact seam a plugin registry
grows from.

## Evidence from Phase 0

- `docs/02-discovery/discovery-moves.md`: five moves, each with the experiment that earned it, and
  an explicit statement that "the build takes exactly the proven moves and no others."
- Exp 4 contributed **no new move** — "evidence that the move-set is stabilising / bounded — the
  strongest single argument for building a small, depth-limited engine rather than an open-ended
  traversal."
- ADR-003 rejects a fuller engine as "infrastructure ahead of proof."
- The fifth move (return-empty/flag) is not an extractor of its own beyond premises: it is the
  *absence* of speculative fetching plus `AssumptionWriter` — see
  [ADR-A009](ADR-A009-premise-catalogue.md).

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Plugin/extension registry for moves** | Excluded by the brief; and a bounded, evidence-earned set does not need an extension mechanism. It would let an unproven move enter without an experiment. |
| **One generic "reference extractor" driven by config rules** | Config-driven rules are a plugin system with extra steps, and would make the tool's behaviour non-obvious from its source — against P8's inspectability. |
| **Collapse the four extractors into one class** | Loses per-move attribution. The bundle's `assertion_kind` exists so the scored run can say which move paid off; that measurement needs the moves separated in code as well as in output. |
| **Add a fifth extractor now for config/schema** | X3: n=1. Handled by flag until a second commit confirms. |

## Consequences

- If Experiment 5 needs a move the four extractors cannot express, the tool produces a visibly
  incomplete bundle. That is the intended failure mode: it sends the question back to the research
  repository rather than being patched away.
- Adding a move requires: a new experiment, a requirement entry, an ADR, and a `Wiring` change —
  four visible steps, in that order.
