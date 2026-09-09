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
| 7 | [decisions/](decisions/) | ADR-A001…A023 — the architecture decisions, with evidence and rejected alternatives |
| 8 | [evidence-gaps.md](evidence-gaps.md) | What the research repository does *not* settle, the architecture's response, and the fourteen recorded architectural assumptions |
| 9 | [REVIEW-freeze-01.md](REVIEW-freeze-01.md) · [02](REVIEW-freeze-02.md) · [03](REVIEW-freeze-03.md) · [04](REVIEW-freeze-04.md) · [05](REVIEW-freeze-05.md) · [06](REVIEW-freeze-06.md) | The freeze reviews: findings, corrections applied, the freeze verdict, the two implementation blockers closed, the ACP-01 patch, the empty-result distinction, and the per-resolver failure premise |

## Status

**ADR-A023 accepted** — a body that keeps its signature but changes what *kind* of value it
returns is a caller question. `return $query->get();` becoming `return $query->first();` inside
`getAll(): Collection` moves no parameter, narrows no declared type, and breaks every caller that
iterates the result — none of which the diff shows. A sixth `assertion_kind`,
`changed_return_contract`, raised by a fifth extractor when a removed and an added `return` end in
names a **closed cardinality table** puts in different classes. It resolves through the existing
`CallerResolver` — the same bounded, non-recursive grep, so the **resolver count stays three** —
and is banded and dropped exactly as `ChangedSignature` is. `bundle_version` stays `1`: adding a
kind is additive. Four receiver-dependent names (`find`, `findOrFail`, `chunk`, `toArray`) were
removed from the table rather than guessed, and the one that remains against the strict standard
(`get`) is recorded in the ADR rather than hidden. The move also made explicit a contract
the tool had always relied on implicitly: **`--repo` is the post-image tree**, now stated in
`03-interfaces.md` §1. Pinned by `ChangedReturnContractHunkOffsetTest`, which drives real
`git diff` output. The ADR records that ADR-A003's four-step gate was satisfied **out of order** —
the `Wiring` change preceded this ADR.

**ADR-A022 accepted** — an experiment resolves source against the tree of **the commit it is
reviewing**, obtained as a detached `git worktree`, never against the repository's current HEAD. This
corrects a **measurement** defect, not a production one: M18 and M19 passed `--repo` pointing at HEAD
while `--diff` was a historical commit, so for a commit whose files had since moved the tool read the
wrong tree or none at all — 6 of M18's 46 commits carry an `unreadable path` diagnostic for it, and
it compromised one M19 task outright. On that task the correction moves the bundle from 2 items / 221
tokens to 4 / 449; across M20's corpus the corrected harness produces materially larger bundles, so
**M18's and M19's figures are understated rather than inflated**. Pinned by
`tests/Acceptance/HarnessResolvesAtCommitTreeTest.php`. No production file changed.

**ADR-A021 accepted** — two bundle items are the same item when **every field a reviewer can see** is
the same: lever, reason, assertion kind, provenance path, member and line span, and payload. When two
items are the same item, the bundle carries it once. Every field in that unit is forced by a keyed
row of `experiment-16`, because each cheaper identity destroys one: `(path, member, span)` collapses
`ControllerA::helper` reached as a same-file sibling *and* as a named reference — three items in two
different `ItemPriority` bands — and payload text collapses a **flag**, taking one of two lines away
from the reviewer. Flags need no special case: a flag's provenance **is** its origin, so two origins
are two identities. On the fixture 15 → 13 items and 328 → 278 tokens; **on M7's real pull request
nothing changes at all**, because it contains no duplicates.

**ADR-A020 accepted** — when a `Fqcn::member` reference cannot be resolved, the class's **surface** is
fetched instead of nothing, provided all three hold: the map places the class, the located path is
**project source**, and nothing in that class's own file accounts for the member. This is M13's
boundary **B4**, applied at resolution time, and it closes gap **G1** — Experiment 1's model-surface
move had never fired on real Laravel code, which reaches a model through `Setting::updateOrCreate(...)`
and never through `new Setting`. The unresolved-member flag is **retained** beside the surface, not
replaced by it: `Package::activatte` — a typo — satisfies the same three conditions as
`Package::create`, and ADR-A016 established that nothing available separates them, so replacing the
flag would report a typo as satisfied context (P10, M0 risk R1). On M7's real pull request the bundle
moves to **18 items / 606 tokens**, recall **1/4 → 3/4**, still **zero** vendor items.

**ADR-A019 accepted** — a fetched slice is withheld when, and only when, the diff shows that exact
span **in full**: the declaring file was created, or one changed region contains the span entirely.
"The path appears in the diff" is explicitly **not** sufficient — a modified file shows only its
hunks, and `experiment-12` keys that counter-case. On M7's real pull request the bundle is now
**11 items / 218 tokens**, down from 1231 when the tool was first measured end to end.

**ADR-A018 accepted** — a file the diff **created** yields no own-file assertions: every line of it is
already in front of the reviewer, so fetching its `use` block, enclosing member or siblings hands the
diff back as context (ADR-A005). Non-PHP files are no longer read as PHP, and the `<?php` open tag is
no longer reported as a missing import. On M7's real pull request the bundle falls from **26 items /
1231 tokens to 14 items / 536 tokens** with **zero items added** and every flag intact.

**ADR-A017 accepted** — a diagnostic now names the state that was observed. `pathFor()` returns null
for two different states, and the tool reported both as `missing PSR-4 entry`; that is **false** when
the prefix exists and only the file is missing. State B gets `class file not found`; state A keeps its
message, because it was already true. The bundle is byte-identical — no item, lever, premise,
provenance, ordering, token count or schema field changed.

**ADR-A016 accepted** — open question **OQ6** is closed as **unresolvable on the current evidence**.
`Package::create`, `Package::activatte` and `Package::totallyUnknownThing` are identical in every one
of the seven facts available when resolution gives up, yet their correct answers differ — so no rule
over those facts can classify them. M0's proposed Eloquent rule **L1** is now known to be unsafe
rather than merely unbuilt. **Nothing changed.**

**ADR-A015 accepted** — ADR-A010's revisit trigger was **evaluated and declined**. M7's E5.4 satisfies
its wording, but the one-hop relaxation it licenses reaches neither E5.4 (three files away, and
blocked by *extraction* rather than depth) nor any of the nine measured false positives (all Eloquent
dynamic dispatch at three or more hops). Across an entire real application, one hop into project code
fixes **0** of 190 static call sites. The boundary stands; **nothing changed**.

**ADR-A014 accepted** — Composer's generated `vendor/composer/autoload_psr4.php` is read as
**location** metadata: **parsed, never executed**, appended after the project's own map so the project
keeps precedence, with ownership still decided by the path. A realistic Laravel application — whose
`composer.json` declares only `App\` — now reaches the settlement behaviour A011–A013 built, without
any framework source entering a bundle. A bare checkout, or absent/malformed metadata, degrades to
exactly the previous behaviour.

**ADR-A013 accepted** — a member declared by an installed dependency is **settled, not fetched**: no
bundle item, one diagnostic citing its file and line. The evidence is `context-types.md` type 2,
*"Named collaborator code (application code, depth one)"*, which covers methods explicitly. A member
that is **not** really there still flags, so a typo against a real package is never swallowed. With
this the ownership rule is uniform across classes and members, and no framework source reaches any
bundle.

**ADR-A012 accepted** — the bare-class **surface** move applies to the project's own classes. A class
the PSR-4 map places inside Composer's dependency directory is settled from its path alone: no bundle
item, one cited diagnostic, and its file is never opened. This closes the largest precision defect the
M1 fixture recorded — one framework return type had expanded into 115 slices and 13,426 tokens — with
no new assertion kind, premise, lever, bundle field or port.

**ADR-A011 accepted** — the two recognition rules ADR-A010 left reachable are implemented: a facade's
`@method static` tag, and Eloquent's `scope<Name>` convention. A framework-known reference is a
**successful negative** (freeze review 05): no bundle item, one cited diagnostic, and the framework's
own source is never fetched. A scope resolves to the *project's* own member and is fetched normally.
No assertion kind, premise, lever, bundle field or port was added — `Ports` stays at five.

**ADR-A010 accepted** — resolution opens at most **one file beyond the changed file**; inheritance
(`extends`, `use <Trait>`) and annotation (`@mixin`) chains are **not** followed, even to verify that
a member exists. This resolves open question OQ1 from the M0 framework-knowledge research and closes
no other question. It separates the two guarantees that "depth one" had been carrying as one phrase:
**D1**, resolved sources never become new input (P4/X1 — *never*), and **D2**, at most one file
beyond the changed file (X2 — *not yet*, now an under-build with a named trigger in
[evidence-gaps.md](evidence-gaps.md) §4). No module, port, capability, or contract changed; the
implementation at `v0.1.0` already conforms.

**Implementation ready** at [REVIEW-freeze-06.md](REVIEW-freeze-06.md) — patch: a seventh premise,
`caller-search-failed`, so each lookup that can fail has its own true statement; premises are never
shared across resolvers. Previously at [REVIEW-freeze-05.md](REVIEW-freeze-05.md) — patch: an empty resolver
result is split into lookup *failure* (flag, P10) and successful *negative* (no item, one diagnostic);
no seventh premise. Previously at [REVIEW-freeze-04.md](REVIEW-freeze-04.md) — patch: one `AssertionKind`
case (`SameFileReference`) so the five kinds partition the five moves one-to-one, plus one provenance
clarification. Previously implementation-ready at [REVIEW-freeze-03.md](REVIEW-freeze-03.md). Frozen at
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
