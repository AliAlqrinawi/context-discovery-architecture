# ADR-A006 · Reverse-caller is a bounded in-process grep, not a call graph and not a subprocess

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R3, P2, P8, P9, P10

## Why this decision exists

Reverse-caller is the recurring, high-severity move and the sharpest A-vs-C differential — and it
is also the expensive one, the reason the fetch/flag split exists. The architecture must pin its
mechanism precisely, because this is the module most likely to grow into a maintained call graph.

## Decision

`Adapters\Search\ScopedGrepCallSiteSearch` scans `*.php` files under a scope prefix
(default `app/`) for occurrences of `<methodName>(` and returns matching lines with path and line
number. It is:

- **in-process** — no subprocess, no shelling out to `grep`/`rg`;
- **bounded** — `--max-call-sites` (default 20) per changed signature; the excess is recorded as a
  flag, never a silent truncation (P10). The bound is required by P7/P10; its *value* is an
  architectural assumption (AA1), not evidence;
- **ordered** — files are scanned in the lexicographic order `filesUnder()` returns, so the bounded
  subset is identical on every machine (P8);
- **stateless** — no index, no cache, no persisted graph (P9);
- **used only for `ChangedSignature` assertions.** The deep "is there a transaction somewhere up
  the stack?" question is **flagged**, never searched (P2).

## Evidence from Phase 0

- R3: "Given a changed method signature, find its callers (**a crude grep over `app/` is
  sufficient**) … Crude grep only — a real call graph is not yet earned."
- Exp 4's minimum context: "A caller search for `reactivate(` across `app/` — a grep, not a file
  fetch."
- `fetch-vs-flag.md`: the transaction question's value "is captured by the flag … and keeps the
  engine out of reverse-graph machinery entirely."
- ADR-002 explicitly bans "turning reverse-caller into a maintained call graph."
- ADR-003 rejected dropping R3 entirely: it is the finding type that most distinguishes the
  conditions. So it must exist — in its cheap form.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **A maintained reverse call graph / symbol index** | Banned by ADR-002 and P9; unearned by evidence. |
| **Shell out to `grep` or `ripgrep`** | Output would depend on the host's installed tools and locale — against P8 determinism — and adds an external requirement to a zero-dependency tool. |
| **Unbounded call-site collection** | A common method name could flood the budget and crowd out higher-priority items; the bound plus a flag keeps the cost visible. |
| **Also grep for the deep transaction question** | `fetch-vs-flag.md` makes it explicit that the flag is the intended lever there; searching would be the expensive machinery the model exists to avoid. |
| **Semantic/AST-accurate call resolution** | Depth-one, evidence-bounded scope does not need it; R3 says a crude grep suffices. Accuracy improvement is an optimisation before measurement. |

## Consequences

- Call sites may include false matches (a same-named method on another class). This is a known,
  documented imprecision, recorded per item so the scored run can measure it — exactly the kind of
  thing the first A-vs-C run should quantify before anyone builds a graph.
- If the scored run shows grep imprecision is the limiting factor, that is the trigger to revisit —
  recorded in [evidence-gaps.md](../evidence-gaps.md).
