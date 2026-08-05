# 05 · Traceability

Nothing in this architecture exists without a line in this table. If a future module cannot be
added to it, that is the signal to stop.

## 1 · Requirement → module

| Req (research) | Capability | Modules that implement it | Experiment(s) |
|---|---|---|---|
| **R1** | Diff parsing + full changed-file load | `Discovery\Parsing\UnifiedDiffParser`, `Adapters\Filesystem\LocalSourceRepository`, `Domain\Diff\*`, `Discovery\Extraction\OwnFileAssertionExtractor` (`SameFileSymbolAbsence`, `SameFileReference`), `Discovery\Resolution\OwnFileResolver` | 1 |
| **R2** | Depth-one named-reference resolution via PSR-4, minimal slice | `Discovery\Extraction\NamedReferenceAssertionExtractor`, `Discovery\Resolution\NamedReferenceResolver`, `Adapters\Autoload\ComposerPsr4ClassLocator`, `Adapters\Php\TokenizerMemberSlicer` | 1, 4 |
| **R3** | Reverse-caller lookup for changed signatures (grep) | `Discovery\Extraction\ChangedSignatureAssertionExtractor`, `Discovery\Resolution\CallerResolver`, `Adapters\Search\ScopedGrepCallSiteSearch`. Signature changes only (Exp 4); a caller question that is not a signature change — Exp 1's transaction — is flag-type per `fetch-vs-flag.md` and counts as a catch per the spec's success criteria | 1, 4 |
| **R4** | Bundle with reason + lever per item, token accounting | `Domain\Bundle\*`, `Assembly\BundleAssembler`, `Assembly\TokenEstimate`, `Adapters\Serialization\*` | method; 1, 3 |
| **R5** | Flag path for expensive/unknowable dependencies | `Discovery\Extraction\UnverifiablePremiseAssertionExtractor`, `Discovery\Lever\LeverPolicy`, `Discovery\Lever\PremiseCatalogue` (six premises, one literal trigger each — [ADR-A009](decisions/ADR-A009-premise-catalogue.md)), `Discovery\Flagging\AssumptionWriter` | 1, 3 |

## 2 · Discovery move → module → evidence

| Move (`docs/02-discovery/discovery-moves.md`) | Lever | Module | Earned by |
|---|---|---|---|
| Read the changed file's own surroundings | Fetch | `OwnFileAssertionExtractor` + `OwnFileResolver`; kinds `SameFileSymbolAbsence`, `SameFileReference` | Exp 1 — missing `Log` import; `upsertFromPlaid` sibling |
| Fetch a named collaborator / model / enum (depth one) | Fetch | `NamedReferenceAssertionExtractor` + `NamedReferenceResolver`; kind `NamedReference`, cross-file only | Exp 1 — `PlaidAccount`; Exp 4 — `PlaidClient::createLinkToken`, `PlaidItemStatus` |
| Reverse-caller lookup | Fetch (bounded) / Flag (deep) | `ChangedSignatureAssertionExtractor` + `CallerResolver` for a changed signature; the deep case (Exp 1's transaction) → `surrounding-transaction` premise, never a search | Exp 1 — transaction (flag-type); Exp 4 — `reactivate` signature (fetch-type) |
| Fetch config / migration | **Flag only in Phase 1** | `PremiseCatalogue` (`atomic-lock-store`, `schema-index-support`, `data-state-after-behaviour-change`) | Exp 3 — n=1, so X3 defers the resolver |
| Return empty / flag rather than fetch | Flag / nothing | `LeverPolicy`, `AssumptionWriter`, and the *absence* of speculative fetching | Exp 1, 2, 3 |
| ~~Forward import-following~~ | — | **no module** — banned structurally (stage 5 never re-enters stage 3) | Exp 1, disproven |

## 3 · Context type → handling

| Type (`docs/02-discovery/context-types.md`) | Phase 1 handling | Module |
|---|---|---|
| 1 · Same-file | Fetch | `OwnFileResolver` — kinds `SameFileSymbolAbsence`, `SameFileReference` |
| 2 · Named collaborator (depth 1) | Fetch | `NamedReferenceResolver` — kind `NamedReference` |
| 3 · Caller | Fetch bounded grep; flag the deep question | `CallerResolver`; `AssumptionWriter` |
| 4 · Schema (migrations) | **Flag** (X3) | `PremiseCatalogue.schema-index-support`, `.data-state-after-behaviour-change` |
| 5 · Configuration | **Flag** (X3) | `PremiseCatalogue.atomic-lock-store` |
| 6 · Runtime / environment | Flag only | `AssumptionWriter` |

## 4 · Principle → structural guarantee

| Principle | How the architecture guarantees it (not "remembers" it) |
|---|---|
| P1 assertion-resolution | `Domain\Assertion` is the only type the pipeline's middle stages accept; there is no "related files" type anywhere. The five kinds partition the five moves one-to-one, so a kind names its resolver and its drop band with no extra field |
| P2 two levers | `LeverPolicy` is the single decision point; `Lever` is a required field on every item |
| P3 depth one | Resolved slices are payload; stage 5 has no path back to stage 3 |
| P4 no forward-following | No module reads a resolved file's own `use` block; `ImportFollower` does not exist |
| P5 self-justifying items | `BundleItem` cannot be constructed without a non-empty reason and a lever |
| P6 judges nothing | No severity/score type; no LLM port; `ItemPriority` is used only for drops and is documented as not a relevance signal |
| P7 budget + visible drops | `--budget` required; `BudgetEnforcer` writes every drop into `dropped[]`; the field is never omitted; the drop order is stated in [01-architecture §3.4](01-architecture.md) |
| P8 deterministic | Fixed stage order, fixed item ordering, lexicographically ordered `filesUnder()` and call sites, no clock, no randomness, no network, no parallelism |
| P9 free local inputs | Four inputs only; no index, no cache directory, no persistence adapter |
| P10 fail to flagging | Every resolver returning nothing routes to `AssumptionWriter`; unreadable path ⇒ flag + diagnostic |

## 5 · Exclusion → enforcement

| Excluded | Enforced by |
|---|---|
| X1 forward import-following | No module; pipeline shape makes it impossible without adding a stage |
| X2 depth > 1 / transitive | Same; also no queue or worklist type exists to hold pending references |
| X3 config/migration resolver | Not in the module list; the capability appears only as premise-catalogue flags |
| X4 severity / review | No severity type, no comment type, no ranking beyond drop priority |
| X5 in-tool LLM | No HTTP client, no API key input, zero runtime dependencies |
| SaaS / server | No HTTP entrypoint; the only entrypoint is `bin/context-discover` |
| AI agents | No loop, no planner, no tool-calling surface; nine fixed stages |
| Event-driven | No dispatcher, no listener, no bus; stages pass values |
| Plugin system | Extractors and resolvers are a closed set constructed in `Cli\Wiring` |
| Multi-language | PHP/Laravel only: PSR-4 map input, PHP tokenizer slicer. Adding a language means a new architecture phase, not a new adapter |
| Optimisation before measurement | No cache, no index, no parallelism, no early-exit heuristic; the token estimator is deliberately naive and documented as such |

## 6 · New architecture decisions → evidence

| ADR | Decision | Primary evidence |
|---|---|---|
| [A001](decisions/ADR-A001-php-cli-zero-dependencies.md) | PHP 8.2 CLI, single command, zero runtime dependencies | Inputs are the PSR-4 map + PHP files (spec); Laravel domain; P8/P9 |
| [A002](decisions/ADR-A002-pipeline-over-events.md) | Pure pipeline behind ports; no events, no plugins | Nine ordered responsibilities (spec); P8; acceptance test is per-stage checkable |
| [A003](decisions/ADR-A003-closed-move-set.md) | Four extractors / three resolvers as a **closed** set | Move-set bounded by Exp 4; ADR-003 "no plugin"; X2 |
| [A004](decisions/ADR-A004-tokenizer-slicing.md) | Slice members with PHP's built-in tokenizer, no AST library | "minimal slice, ideally one method" (spec); zero-dependency P9 |
| [A005](decisions/ADR-A005-slices-not-files.md) | Full changed-file text is an analysis input; only slices enter the bundle | Exp 2 must yield an almost-empty bundle; Exp 1's minimum-context list |
| [A006](decisions/ADR-A006-grep-not-graph.md) | In-process scoped grep for callers, bounded, no subprocess | R3 "crude grep is sufficient"; P8/P9 |
| [A007](decisions/ADR-A007-token-estimate.md) | One documented characters-per-token ratio constant, not a tokenizer; the ratio recorded as an assumption (AA4) | P7 needs accounting; "no optimisation before measurement" |
| [A008](decisions/ADR-A008-required-budget.md) | `--budget` has no default | The research fixes no absolute number — only "dramatically below B" |
| [A009](decisions/ADR-A009-premise-catalogue.md) | Premises are a closed catalogue, not inference | R5's flags come from three named Phase 0 premises; P6 (no judging) |

## 7 · Architectural assumptions

Eight decisions are reasonable but not proven by Phase 0. They are listed as assumptions — not
evidence — in [evidence-gaps.md §5](evidence-gaps.md), recorded by
[REVIEW-freeze-01.md](REVIEW-freeze-01.md).
