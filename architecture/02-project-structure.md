# 02 · Project Structure

## 1 · Directory layout

```
context-discovery/
├── bin/
│   └── context-discover                 # executable entrypoint; argv → Cli\DiscoverCommand
├── composer.json                        # PSR-4: ContextDiscovery\ → src/ ; zero runtime deps
├── README.md                            # how to run it; points at docs/architecture/
├── src/
│   ├── Cli/
│   │   ├── DiscoverCommand.php
│   │   ├── Options.php
│   │   ├── ExitCode.php
│   │   └── Wiring.php
│   ├── Pipeline/
│   │   └── DiscoverContext.php           # the single use case: nine stages, fixed order
│   ├── Domain/
│   │   ├── Diff/{Diff,ChangedFile,ChangedRegion,ChangedMember}.php
│   │   ├── Assertion/{Assertion,AssertionKind,ResolvedAssertion}.php
│   │   ├── Bundle/{Bundle,BundleItem,Lever,Provenance,DroppedItem}.php
│   │   └── Source/{SourceSlice,CallSite}.php
│   ├── Ports/
│   │   ├── SourceRepository.php
│   │   ├── ClassLocator.php
│   │   ├── MemberSlicer.php
│   │   ├── CallSiteSearch.php
│   │   └── BundleWriter.php
│   ├── Discovery/
│   │   ├── Parsing/
│   │   │   └── UnifiedDiffParser.php      # pure, not a port
│   │   ├── Extraction/
│   │   │   ├── AssertionExtractor.php            # facade over the four below, fixed order
│   │   │   ├── OwnFileAssertionExtractor.php     # R1
│   │   │   ├── NamedReferenceAssertionExtractor.php  # R2
│   │   │   ├── ChangedSignatureAssertionExtractor.php # R3
│   │   │   └── UnverifiablePremiseAssertionExtractor.php # R5
│   │   ├── Lever/
│   │   │   ├── LeverPolicy.php                   # the fetch-vs-flag decision rule
│   │   │   └── PremiseCatalogue.php              # closed list; one entry per Phase 0 premise
│   │   ├── Resolution/
│   │   │   ├── OwnFileResolver.php
│   │   │   ├── NamedReferenceResolver.php
│   │   │   └── CallerResolver.php
│   │   └── Flagging/
│   │       └── AssumptionWriter.php
│   ├── Assembly/
│   │   ├── BundleAssembler.php
│   │   ├── ItemPriority.php               # drop order only
│   │   ├── TokenEstimate.php              # pure, not a port
│   │   └── BudgetEnforcer.php
│   └── Adapters/
│       ├── Filesystem/LocalSourceRepository.php
│       ├── Autoload/ComposerPsr4ClassLocator.php
│       ├── Php/TokenizerMemberSlicer.php
│       ├── Search/ScopedGrepCallSiteSearch.php
│       └── Serialization/{JsonBundleWriter,MarkdownBundleWriter}.php
├── tests/
│   ├── Unit/                            # mirrors src/ one-to-one
│   ├── Acceptance/
│   │   ├── ExperimentKeyTest.php         # the Phase 0 acceptance test (06-acceptance.md)
│   │   └── fixtures/            # expected-context.md shipped;
│   │       │                     # diff.patch is operator-supplied
│   │       ├── experiment-01/{diff.patch, expected-context.md}
│   │       ├── experiment-02/{diff.patch, expected-context.md}   # expects an almost-empty bundle
│   │       ├── experiment-03/{diff.patch, expected-context.md}
│   │       └── experiment-04/{diff.patch, expected-context.md}
│   └── Fakes/                           # in-memory port doubles; no filesystem in unit tests
└── docs/
    └── architecture/                    # copy of this contract; its header names the
                                         # origin and the freeze review it was frozen at
```

Nothing else at the root. No `config/`, no `storage/`, no `var/`, no cache directory — the tool
holds no state between runs (P8, P9).

## 2 · Module organisation

Seven groups, each with one job:

| Group | Contains | May know about |
|---|---|---|
| `Domain` | Value types and their invariants | nothing |
| `Ports` | Interfaces the inner code calls | `Domain` |
| `Discovery` | The validated moves as pure logic | `Domain`, `Ports` |
| `Assembly` | Bundle construction, budget | `Domain`, `Ports` |
| `Pipeline` | Stage ordering (the use case) | `Domain`, `Ports`, `Discovery`, `Assembly` |
| `Adapters` | I/O, formats, PHP tokenizing | `Domain`, `Ports` |
| `Cli` | argv, wiring, exit codes, output channel | everything |

One file per class. One class per concept. No abstract base classes, no traits shared across
groups, no static registries.

## 3 · Domain boundaries

Three domains, deliberately separated because they are checked separately against the keys:

1. **Diff domain** — "what changed". Knows files, regions, member signatures. Knows nothing about
   PHP semantics or about context.
2. **Assertion domain** — "what the change claims but cannot prove". The centre of gravity
   (P1/ADR-002). Knows the four assertion kinds and nothing about how they are resolved.
3. **Bundle domain** — "what we are handing the reviewer". Knows items, reasons, levers,
   provenance, budget, drops. Knows nothing about PHP, diffs, or the filesystem.

Boundary rules:

- Diff types never reference Assertion or Bundle types.
- Assertion types reference a diff *origin* (path + region), never a `ChangedFile` object graph.
- Bundle types reference `Provenance` (path, member, line span) — plain data, not Diff types.
- Nothing in any domain performs I/O, formatting, or token counting.

## 4 · Dependency direction

```
Cli ──▶ Pipeline ──▶ Discovery ──▶ Ports ──▶ Domain
            │            │                     ▲
            └──▶ Assembly ┘                     │
Adapters ─────────────────────────────────────── (implement Ports, use Domain)
Cli ──▶ Adapters                       (construction only, in Wiring)
```

Rules, enforceable by review or a static check:

1. Dependencies point **inward**. `Domain` imports nothing from the project.
2. `Discovery`, `Assembly`, and `Pipeline` may reference `Ports` interfaces only — **never** an
   `Adapters` class.
3. `Adapters` may reference `Ports` and `Domain` only — never `Discovery`, `Assembly`, or `Pipeline`.
4. `Cli\Wiring` is the single place where a concrete adapter is named. Adding an adapter must
   change exactly one file.
5. No cycles anywhere. No module imports its own consumer.
6. No global functions, no singletons, no service locator, no runtime reflection to find classes.

Consequence: the tool can be exercised entirely with in-memory fakes, which is what makes the
Phase 0 acceptance test cheap and hermetic.

## 5 · Naming conventions

**Code**

- PHP 8.2, `strict_types=1`, PSR-12, PSR-4 (`ContextDiscovery\` → `src/`).
- Classes `PascalCase`; methods and properties `camelCase`; enum cases `PascalCase`.
- Interfaces are named for the capability, without an `Interface` suffix (`SourceRepository`, not
  `SourceRepositoryInterface`). *Repository* here means **the repository under review**; the
  persistence-pattern ban is unaffected. Implementations carry the qualifier that distinguishes them
  (`LocalSourceRepository`, `ComposerPsr4ClassLocator`, `ScopedGrepCallSiteSearch`).
- Value objects are immutable: `readonly` properties, constructor-only assignment, no setters.
- Extractors end in `AssertionExtractor`; resolvers end in `Resolver`; writers end in `Writer`;
  the one decision object is a `Policy`.
- Vocabulary is the research repository's vocabulary, verbatim: *assertion*, *lever*, *fetched*,
  *flagged*, *reason*, *provenance*, *bundle*, *dropped*, *premise*, *call site*, *slice*. No
  synonyms — no "relevance", no "score", no "candidate", no "context engine".

**Output fields** — `snake_case` in JSON (`used_tokens`, `budget_tokens`, `bundle_version`) so the
serialised bundle reads like the illustrative shape in the implementation spec.

**Tests** — `tests/Unit/` mirrors `src/` path for path; acceptance fixtures are named
`experiment-0N` to match the research repository's filenames exactly, so a reader can put the two
side by side.

**Forbidden names** (they signal scope escape): `*Engine`, `*Agent`, `*Plugin`, `*EventBus`,
`*Listener`, `*Cache`, `*Index`, `*Graph`, `*Score`, `*Severity`, `*Reviewer`, `*Client`.
