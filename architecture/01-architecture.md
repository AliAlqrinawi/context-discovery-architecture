# 01 · Architecture

## 1 · High-level architecture

One process. One command. Nine ordered stages, all synchronous, all deterministic. A batch
transformation, not a service:

```
diff (+ repo on disk)  ──▶  context-discover  ──▶  context bundle (JSON | Markdown)
```

The shape is a **pure pipeline behind ports**:

- A thin **CLI** shell parses options, wires collaborators by constructor, and writes output.
- A single **use case** (`DiscoverContext`) runs the nine spec responsibilities in fixed order.
- **Discovery** modules are pure logic: they turn a diff into *assertions*, choose a *lever* per
  assertion, and resolve or flag it.
- **Assembly** builds the bundle, estimates tokens, enforces the budget, records drops.
- **Adapters** are the only code that touches the filesystem, `composer.json`, PHP source text,
  or the output stream. They are reached exclusively through **ports** (interfaces).

Why this shape and not another: determinism and inspectability are requirements (P8), the inputs
are free and local (P9), and the acceptance test is "run it on four commits and compare the bundle
to a hand-written key" (`implementation-spec.md`). A pipeline of pure stages behind three
filesystem ports is the smallest structure that makes each stage independently checkable against a
key. See [ADR-A002](decisions/ADR-A002-pipeline-over-events.md).

## 2 · System boundaries

**Inside the boundary** (this architecture owns):

1. Unified-diff parsing into changed files, regions, and changed member signatures.
2. Assertion extraction for the four validated assertion kinds.
3. The fetch-vs-flag lever decision.
4. Depth-one reference resolution via the PSR-4 map, and minimal member slicing.
5. Bounded caller search (in-process grep) over a configured scope.
6. Assumption (flag) text generation from a closed catalogue.
7. Bundle assembly, token accounting, budget enforcement, drop recording.
8. Two serialisations of the bundle: JSON (machine, stable) and Markdown (paste alongside the diff).

**Outside the boundary** (consumed, never owned):

- Git. The diff arrives as a file or on stdin; the tool never invokes Git, never resolves refs,
  never touches the index or worktree state beyond reading files.
- The target repository. Read-only, on local disk.
- `composer.json`'s PSR-4 autoload map. Read-only input (P9).

**Outside the boundary** (downstream, deliberately absent):

- The reviewer. No LLM call, no prompt template, no API client (X5, P6).
- Scoring A vs C. Human work, Phase 2 (X4).
- Any server, queue, daemon, scheduler, watcher, or hook. There is no runtime beyond the command.

**Trust boundary:** the tool reads only inside `--repo`. Path resolution rejects any resolved path
that escapes the repository root; unreadable or out-of-scope paths become **flags**, never silent
omissions (P10).

## 3 · Internal modules and responsibilities

Namespace root: `ContextDiscovery\`. Layers are listed inward-to-outward; dependencies point
inward only ([02-project-structure.md](02-project-structure.md) §4).

### 3.1 Domain (no dependencies)

| Module | Responsibility | Traces to |
|---|---|---|
| `Domain\Diff` | Immutable value types: `Diff`, `ChangedFile`, `ChangedRegion`, `ChangedMember` (name + old/new signature). Knows nothing about diff *text*. | R1 |
| `Domain\Assertion` | `Assertion` (kind, subject, origin region, human-readable claim) and `AssertionKind`: `SameFileSymbolAbsence`, `NamedReference`, `ChangedSignature`, `UnverifiablePremise`. The core domain concept — the diff's *claims*, not its files. Plus `ResolvedAssertion` — assertion + lever + either slices or a statement — the single value handed from resolution to assembly. | P1, ADR-002 |
| `Domain\Bundle` | `Bundle` (ordered items, budget, used tokens, drops), `BundleItem` (payload, reason, lever, provenance), `Lever` enum (`Fetched`\|`Flagged`), `Provenance` (path, member, line span), `DroppedItem` (reason, note). Enforces the invariant: **no item without a reason and a lever**. | R4, P5 |
| `Domain\Source` | `SourceSlice` (path, member, line span, text) — the minimal fetched payload. | R2 |

### 3.2 Ports (interfaces only, no logic)

Five ports, one per side effect. Pure text transformations are **not** ports: diff parsing and token
estimation perform no I/O and have one implementation each, so they are plain classes
(freeze review O1/O2).

| Port | Responsibility | Traces to |
|---|---|---|
| `Ports\SourceRepository` | Read-only, root-scoped file access: `text(path)`, `exists(path)`, `filesUnder(prefix, extension)`. | R1, P9 |
| `Ports\ClassLocator` | Fully-qualified class name → file path, via the PSR-4 map. Depth-one only, no index. | R2, P9 |
| `Ports\MemberSlicer` | File text + member name → `SourceSlice`; also `useBlock(text)` and `memberNames(text)`. | R1, R2 |
| `Ports\CallSiteSearch` | Method name + scope prefix → `CallSite[]` (path, line, line text). A bounded grep, never a graph. | R3 |
| `Ports\BundleWriter` | `Bundle` → serialised output. | R4 |

### 3.3 Discovery (pure logic over ports)

| Module | Responsibility | Traces to |
|---|---|---|
| `Discovery\Parsing\UnifiedDiffParser` | Unified diff text → `Domain\Diff`. A pure class, not a port: no I/O, one implementation. Old signatures come from the hunk's removed lines; a rename is treated as two members and is not tracked. | R1 |
| `Discovery\Extraction\AssertionExtractor` | Facade. Runs the four extractors over every changed region in a fixed order and returns a deterministic, de-duplicated `Assertion[]`. | spec resp. 3 |
| `…\OwnFileAssertionExtractor` | Reads the changed file's own text: emits `SameFileSymbolAbsence` for a symbol used in the region but missing from the `use` block, and `NamedReference` (same-file) for sibling members the region calls. | R1; Exp 1 (missing `Log`, `upsertFromPlaid`) |
| `…\NamedReferenceAssertionExtractor` | Emits `NamedReference` for classes/members the region names and that are not defined locally — model, client method, enum. Depth one; **does not** follow the resolved file's own references (P3). | R2; Exp 1, 4 |
| `…\ChangedSignatureAssertionExtractor` | Compares old/new signatures of members touched by the diff; emits `ChangedSignature` when arity or parameter shape changed. | R3; Exp 4 (`reactivate`) |
| `…\UnverifiablePremiseAssertionExtractor` | Emits `UnverifiablePremise` from a **closed, evidence-derived catalogue** of premises a file cannot settle: surrounding-transaction, atomic-lock-store, schema-index-support, data-state-after-behaviour-change. Each has a single literal **trigger** — a premise is emitted if and only if its trigger is present ([ADR-A009](decisions/ADR-A009-premise-catalogue.md)); there is no inference path. Adding a premise, or a trigger, requires a new experiment. | R5; Exp 1, 3 |
| `Discovery\Lever\LeverPolicy` | One pure function implementing `fetch-vs-flag.md`'s decision rule: named + single + depth-one on disk → `Fetched`; expensive (reverse-graph) or unknowable (runtime/data) → `Flagged`. No heuristics beyond that rule. | R5, P2 |
| `Discovery\Resolution\OwnFileResolver` | Fetches the region's own `use` block, the enclosing member, and named sibling members. | R1 |
| `Discovery\Resolution\NamedReferenceResolver` | Locates the class (`ClassLocator`), slices the single named member or the model/enum surface (`MemberSlicer`). Unresolved → returns nothing and hands the assertion back for flagging (P10). | R2 |
| `Discovery\Resolution\CallerResolver` | For a `ChangedSignature`, greps call sites within the configured scope and returns the matching lines. Bounded: scope prefix, max call sites, no recursion. | R3 |
| `Discovery\Flagging\AssumptionWriter` | Renders one flagged assertion into a one-line `ASSUMPTION: …` payload plus its reason. Text templates only, one per catalogue premise. | R5 |

#### Reference forms recognised by `NamedReferenceAssertionExtractor`

R2's recall is defined by this closed list. Each form is required by a finding; any other form
produces **no** assertion — no inference, no guessing (ADR-A003).

| Form | Example from the evidence | Earned by |
|---|---|---|
| `Name::member` — static or enum member access | `PlaidItemStatus::REVOKED` | Exp 4 |
| `$property->method(` where the property's declared type resolves through the file's `use` block | `$this->plaidClient->createLinkToken(` | Exp 4 |
| `$this->method(` — same-class sibling | `upsertFromPlaid` | Exp 1 |
| A class name in a `new`, type, or static-call position, resolved through the `use` block | `PlaidAccount` model surface | Exp 1 |

Resolution is depth one: the resolved file's own references are never read (P3, P4).

#### Premise triggers

The parallel closed list for `UnverifiablePremise` — six premises, one literal trigger each — is
specified in [ADR-A009](decisions/ADR-A009-premise-catalogue.md). Two properties matter
architecturally: every trigger reads only the changed region and the changed file's own text (R1's
existing load, no new input), and nothing outside the list can produce a premise. That is what keeps
Experiment 2's bundle almost empty — no trigger fires on it — and what makes Experiment 4 the
precision guard for the transaction trigger.

#### The caller is fetched only for a changed signature

`CallerResolver` runs for `ChangedSignature` assertions and nothing else. A caller question that is
*not* a signature change — Experiment 1's "is this wrapped in a transaction?" — is **flagged**, never
searched: `fetch-vs-flag.md` names it the canonical flag case, and the implementation spec's success
criteria count "flags as a catch for the flag-type findings (e.g. the transaction assumption)". The
acceptance expectations mark items accordingly ([06-acceptance.md](06-acceptance.md) §1).

### 3.4 Assembly

| Module | Responsibility | Traces to |
|---|---|---|
| `Assembly\BundleAssembler` | Turns resolved slices and flags into `BundleItem`s, attaching reason, lever, provenance. Rejects any item lacking a reason. | R4, P5 |
| `Assembly\ItemPriority` | The fixed, documented priority order used only for budget drops. Not a relevance score (X4). | P7 |
| `Assembly\BudgetEnforcer` | Sums estimated tokens; while over budget, drops the lowest-priority item and records it in `dropped` with its reason. Never drops silently. | R4, P7 |

#### `ItemPriority` — the drop order

Used **only** by `BudgetEnforcer`, and only for dropping. It is not a relevance signal (P6, X4).
Priority 1 is never dropped; priority 4 is dropped first.

| Priority | Items | Why here |
|---|---|---|
| 1 · never dropped | Flagged items | Cost is near-zero, and a silent omission is indistinguishable from "nothing needed" (P10, `fetch-vs-flag.md`) |
| 2 | Own-file slices (`use` block, enclosing member, named siblings) | Free on disk, the substrate every other move builds on (R1, Exp 1) |
| 3 | Changed-signature call sites | The recurring high-severity move and the sharpest A-vs-C differential (R3, Exp 1, 4) |
| 4 · dropped first | Named-reference slices | Valuable and cheap, but the move Exp 2 shows must never be pulled speculatively |

Within a priority band, the largest estimated item is dropped first, so the fewest items are lost.

### 3.5 Adapters

| Adapter | Implements | Notes |
|---|---|---|
| `Adapters\Filesystem\LocalSourceRepository` | `SourceRepository` | Root-scoped, read-only; rejects path escapes. |
| `Adapters\Autoload\ComposerPsr4ClassLocator` | `ClassLocator` | Parses `composer.json` autoload + autoload-dev PSR-4 maps. No class map build (P9). |
| `Adapters\Php\TokenizerMemberSlicer` | `MemberSlicer` | PHP's built-in tokenizer; no AST library ([ADR-A004](decisions/ADR-A004-tokenizer-slicing.md)). |
| `Adapters\Search\ScopedGrepCallSiteSearch` | `CallSiteSearch` | In-process scan of `*.php` under the scope prefix, in the order `filesUnder()` returns; no subprocess ([ADR-A006](decisions/ADR-A006-grep-not-graph.md)). |
| `Adapters\Serialization\JsonBundleWriter`, `MarkdownBundleWriter` | `BundleWriter` | JSON is the stable machine contract; Markdown is the paste-alongside-diff artifact. |

### 3.6 CLI

| Module | Responsibility |
|---|---|
| `Cli\Options` | Parse and validate argv. `--budget` is **required** — the research fixes no number ([ADR-A008](decisions/ADR-A008-required-budget.md)). |
| `Cli\Wiring` | Plain constructor wiring. No container library, no service locator, no auto-discovery. |
| `Cli\DiscoverCommand` | Runs `Pipeline\DiscoverContext`, writes the bundle to stdout or `--out`, diagnostics to stderr, returns an exit code. |
| `Pipeline\DiscoverContext` | The single use case. Nine stages, fixed order, no branching on configuration beyond scope/budget. |

**Module count: 25 classes + 5 interfaces.** Anything larger would be Phase 2 leaking in.

## 4 · Data flow

Fixed order; each arrow is a value, not an event (no bus, no listeners — [ADR-A002](decisions/ADR-A002-pipeline-over-events.md)):

```
1  diff text ─▶ UnifiedDiffParser ─▶ Diff{ChangedFile[]{ChangedRegion[], ChangedMember[]}}
2  ChangedFile[] ─▶ SourceRepository.text() ─▶ full current text per changed file   (analysis input)
3  (Diff, file texts) ─▶ AssertionExtractor ─▶ Assertion[]
4  Assertion[] ─▶ LeverPolicy ─▶ (Assertion, Lever)[]
5a Lever=Fetched  ─▶ OwnFileResolver | NamedReferenceResolver | CallerResolver ─▶ SourceSlice[]
5b Lever=Flagged  ─▶ AssumptionWriter ─▶ assumption text
5c resolution failure ─▶ back to 5b as a flag                                       (P10)
6  (slices, flags) ─▶ BundleAssembler ─▶ Bundle{BundleItem[] with reason+lever+provenance}
7  Bundle ─▶ TokenEstimate + BudgetEnforcer ─▶ Bundle{used_tokens, dropped[]}
8  Bundle ─▶ JsonBundleWriter | MarkdownBundleWriter ─▶ stdout | --out
```

Two properties hold by construction:

- **The full changed-file text is an analysis input, not automatically an output item.** Only
  slices that settle a named assertion enter the bundle — the `use` block, the enclosing member,
  a referenced sibling. This is what keeps Experiment 2's bundle almost empty
  ([ADR-A005](decisions/ADR-A005-slices-not-files.md)).
- **Stage 5 never re-enters stage 3.** Resolved sources are payload, never new input. That is the
  depth-one guarantee, structural rather than configured (P3).

## 5 · Inputs and outputs

### Inputs

| Input | Source | Required | Notes |
|---|---|---|---|
| Diff | `--diff <path>` or `-` (stdin) | yes | Unified diff for one commit under review |
| Repository root | `--repo <path>` | yes | Read-only; all paths resolved inside it |
| Changed-file text | filesystem, derived from the diff | yes | Loaded by the tool, not supplied |
| PSR-4 map | `<repo>/composer.json` | yes | Missing map ⇒ every `NamedReference` becomes a flag, plus one diagnostic |
| Token budget | `--budget <int>` | yes | No default; the research fixes no number |
| Caller scope | `--caller-scope <prefix>` | no, default `app/` | Exp 4's minimum context is "a caller search across `app/`" |

No other input exists. No config file, no environment variables, no network, no state directory.

### Outputs

| Output | Channel | Contract |
|---|---|---|
| Context bundle | stdout | JSON (default) or Markdown — schema in [03-interfaces.md](03-interfaces.md) |
| Diagnostics | stderr | One line per unresolved reference, unreadable path, or missing PSR-4 entry. Never merged into the bundle payload. |
| Exit code | process | `0` bundle produced · `1` usage/input error · `2` repository unreadable |

An empty-but-valid bundle (Experiment 2's correct answer) exits `0`. Emptiness is a result, not a
failure.

## 6 · Interfaces

Public and internal interface signatures are specified in
[03-interfaces.md](03-interfaces.md): the CLI contract and the versioned bundle schema are the
**public** surface; the five ports and the discovery collaborators are **internal** and may change
without notice, since nothing outside the tool depends on them.
