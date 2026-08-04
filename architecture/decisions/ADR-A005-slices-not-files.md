# ADR-A005 · The changed file's full text is an analysis input; only slices enter the bundle

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R1, R4, P7, spec success criteria

## Why this decision exists

R1 requires loading the **full current text** of each changed file, and the spec lists that text
as an *input*. It is easy to read this as "put the whole changed file in the bundle." That reading
would break the precision criterion and Experiment 2's expected result, so the architecture must
state the distinction explicitly.

## Decision

Full changed-file text is loaded for **analysis only**. What enters the bundle from the own-file
move is:

1. the file's `use` block (the structure that makes an absence visible),
2. the member enclosing each changed region,
3. any sibling member the region names.

Nothing else from the changed file is emitted. Every emitted slice still carries a reason.

## Evidence from Phase 0

- The spec: inputs include "the full current text of each changed file … they are free (on disk)";
  outputs are items whose payload is "the minimal source slice (ideally one method or class member,
  not a whole file)". Input and payload are different things.
- Exp 1's minimum context is explicitly "one model and three method bodies. No repository dump,
  nothing close to it."
- Exp 2's correct output is "an almost-empty bundle", and the spec's precision criterion says
  pulling the model "just in case" is "a precision failure, not caution." A whole-file default
  would fail Experiment 2 by construction.
- P7: the budget exists to demonstrate cost far below condition B; whole-file payloads spend it on
  text no finding asked for.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Emit each changed file in full** | Fails Exp 2's near-empty expectation and the spec's "not a whole file"; inflates tokens with material no assertion justifies. |
| **Emit the diff itself inside the bundle** | The reviewer already receives the diff — condition C is "diff **plus** bundle". Duplicating it would double-count tokens and confuse the A-vs-C cost comparison. |
| **Emit a configurable number of context lines around each hunk** | An unmeasured knob; the evidence asks for member-level slices, not radii. |

## Consequences

- Reading a changed file is free; *including* any part of it must be justified by an assertion.
- A reviewer receiving the bundle sees the changed file only through the slices that matter, which
  is what makes the bundle's size comparable to Phase 0's hand-assembled minimum context.
