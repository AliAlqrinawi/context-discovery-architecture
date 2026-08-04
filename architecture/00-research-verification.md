# 00 · Verification of the Validated Research

Read before architecture. Everything below is a restatement of the research repository, with the
file that carries it. No interpretation is added; where the research marks something open, it is
marked open here.

Source read: `README.md`, `ROADMAP.md`, `CHANGELOG.md`, `docs/00-introduction/*`,
`docs/01-phase0/README.md`, `docs/01-phase0/experiment-0{1,2,3,4}.md`, `docs/02-discovery/*`,
`docs/03-phase1/*`, `docs/04-decisions/ADR-00{1,2,3}*`.

## 1 · The project

An investigation into one product hypothesis, documented as research and not as a shipped
product: whether intelligently selected project context can produce a better AI code review than
a raw Git diff while using significantly less context than the whole repository
(`README.md`, `docs/00-introduction/00-project-overview.md`).

Framing is fixed at three review conditions (`docs/00-introduction/02-hypothesis.md`):

| Condition | Reviewer input | Role |
|---|---|---|
| **A** | Raw diff only | Floor |
| **B** | Diff + entire repository | Ceiling / yardstick, never the product |
| **C** | Diff + targeted context bundle | The proposal under test |

Phase 0 (manual validation) is **complete**. Phase 1 (smallest tool that automates the validated
context-assembly step) is **specified, not built**. Phase 2+ is not started (`README.md`,
`ROADMAP.md`).

The studied codebase is a Laravel application with a Plaid banking integration; the examples are
domain-specific, the findings are intended to generalise
(`docs/00-introduction/00-project-overview.md`).

## 2 · The validated hypothesis — and its exact boundary

**Hypothesis (still open):** for a given change there exists a small bundle of off-diff sources,
discoverable from the diff itself, such that C catches materially more real defects than A at a
context cost far below B (`docs/00-introduction/02-hypothesis.md`).

**What Phase 0 actually validated:**

- Answer keys, hand-written before any model ran, for four real commits (ADR-001).
- A classification method that *discriminates*: Experiment 2 was correctly labelled diff-only with
  an essentially empty context-dependent pile (`ROADMAP.md` exit gate, ADR-001 consequences).
- A stable, recurring catalogue of **discovery moves**, each earned by a finding
  (`docs/02-discovery/discovery-moves.md`):
  read-own-file (Exp 1); fetch-collaborator depth one (Exp 1, 4); reverse-caller (Exp 1, 4);
  fetch config/migration (Exp 3); return-empty/flag (Exp 1, 2, 3).
- The **two-lever cost model** — fetch vs flag, chosen by cost (`docs/02-discovery/fetch-vs-flag.md`).
- The **context-type taxonomy**: same-file, named collaborator, caller, schema, configuration,
  runtime/env (`docs/02-discovery/context-types.md`).
- **Reframing**: context discovery is assertion-resolution, not file-finding (ADR-002).
- **One disproof**: naive forward import-following would have missed three of Experiment 1's four
  findings (Exp 1, ADR-002, `CHANGELOG.md`).

**What Phase 0 did not validate (carried as risk, not assumption):** the A/B/C runs were never
executed and scored. So there is *no scored evidence* that supplying context improves a review;
the false-positive rate was never measured; whether a flag captures most of reverse-caller's value
was never measured; and the move-set's boundedness rests on four *polished* commits — the planned
sloppy Experiment 5 was never run (`README.md` caveat, `ROADMAP.md`, `CHANGELOG.md` "Open",
ADR-001, ADR-003).

**Architectural consequence:** none of those open questions may be encoded in the tool
(`docs/03-phase1/requirements.md`, "Assumptions that must stay OUTSIDE the implementation"). The
tool's job is to make the scored run *possible* and *measurable* — nothing more.

## 3 · Phase 1 scope (what this architecture implements)

A single command, **diff in → context bundle out**, deterministic, no LLM inside
(`docs/03-phase1/implementation-spec.md`, ADR-003).

| Req | Capability | Justified by |
|---|---|---|
| **R1** | Diff parsing + full changed-file loading | Exp 1 |
| **R2** | Depth-one named-reference resolution via the PSR-4 autoload map, minimal slice | Exp 1, 4 |
| **R3** | Reverse-caller lookup for changed method signatures — a crude grep over `app/` | Exp 1, 4 |
| **R4** | Bundle items each carrying payload + **reason** + **lever**, with token accounting | Phase 0 method; Exp 1, 3 |
| **R5** | Flag path for expensive (reverse-graph) or unknowable (runtime/env, data-state) dependencies | Exp 1, 3 |

Inputs, fixed by spec: the diff; the full current text of each changed file; the local repository
filesystem; the PSR-4 autoload map from `composer.json`. **Nothing else** — no pre-built index,
no reverse call graph, no network, no LLM.

Output, fixed by spec: an ordered list of items plus a running token count against a budget; each
item carries payload, reason, lever; the bundle records per-item provenance (file path, member
name) and a visible list of items **dropped** for budget. The serialisation format is explicitly
*not* fixed by the spec.

Responsibilities, fixed by spec (nine, in order): parse the diff; load full changed-file text;
extract assertions; apply read-own-file; resolve cheap depth-one references; reverse-caller grep;
flag the expensive/unknowable; attach reason and lever to every item; enforce the budget with
visible drops.

Success criteria: **high recall** against each commit's context-dependent pile (flags count as a
catch for flag-type findings), **efficiency** far below full-repo tokens, and **tolerable
precision** — on Experiment 2 the correct bundle is *almost empty*; pulling the model "just in
case" is a precision failure.

## 4 · Intentionally out of scope

From `docs/03-phase1/requirements.md` and ADR-003:

| Out | Capability | Why | Status |
|---|---|---|---|
| **X1** | Forward import-following / dependency traversal | Disproven — would have missed 3 of Exp 1's 4 findings | **never** |
| **X2** | Transitive / depth > 1 resolution | No finding in four commits needed it | not yet |
| **X3** | Dedicated config/migration resolver | Exp 3 proved config & schema matter, but n=1 → generic flag path instead | not yet |
| **X4** | Severity/quality scoring, or the review itself | The engine assembles; it does not judge | **never** |
| **X5** | Any LLM call inside the tool | Phase 1 is deterministic assembly; the model is downstream | **never** |

Also out, per `docs/03-phase1/implementation-spec.md` non-responsibilities and P9: maintaining an
index or a call graph (reverse-caller is a grep, not a graph).

Out by the standing brief for this architecture work: SaaS, AI agents, event-driven architecture,
plugin systems, multi-language support, and any optimisation before measurement.

Explicitly **not** Phase 1's job, though downstream of it: running the reviews, scoring A vs C,
and the Experiment 5 (sloppy commit) run — which the spec names as the *first* input after build.

## 5 · Things this architecture must not launder into the design

Held out of the code as assumptions under test (`docs/03-phase1/requirements.md`):

1. That supplying the context improves a review.
2. That the false-positive rate stays tolerable.
3. That the move-set is bounded.

Mechanically, this means: no confidence scores, no relevance weighting learned from the keys, no
"probably useful" fetches, and no hard-coded assumption that a flag equals a fetch in value. The
bundle records what it did and why; a human scores whether it helped.
