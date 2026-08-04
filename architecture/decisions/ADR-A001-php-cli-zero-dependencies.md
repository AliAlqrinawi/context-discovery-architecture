# ADR-A001 · PHP 8.2 CLI, one command, zero runtime dependencies

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R1–R5, P8, P9, ADR-003

## Why this decision exists

The implementation spec fixes the tool's inputs — the diff, the changed files' text, the local
repository filesystem, and **the PSR-4 autoload map from `composer.json`** — and its shape: a
single command, diff in, bundle out. Something must decide the host language and dependency
posture before any file is written, because those choices determine whether the tool can slice PHP
members and resolve PSR-4 names without building an index.

## Decision

- **PHP 8.2, CLI only.** One executable, `bin/context-discover`. No framework, no HTTP entrypoint,
  no daemon.
- **Zero runtime dependencies.** Composer is used for autoloading and for a dev-only test runner.
  Nothing is required at run time beyond the PHP standard library.
- **Single target language: PHP/Laravel.** Not a language-agnostic engine.

## Evidence from Phase 0

- The spec names the PSR-4 map as an input "so a class name resolves to a file path without an
  index" (`docs/03-phase1/implementation-spec.md`). PSR-4 resolution and PHP member slicing are
  PHP-specific work; PHP performs them with its own standard library.
- Every experiment is a PHP/Laravel commit: `PlaidAccountService`, `PlaidAccount`,
  `PlaidItemStatus`, `config/cache.php`, a `plaid_items` migration, `routes/api.php`
  (Exp 1, 3, 4). There is no evidence of a second language.
- P9: "no index to maintain, no graph to precompute, no service to call." A dependency-free CLI is
  the smallest thing that satisfies this.
- P8: determinism. Fewer dependencies means fewer sources of version-dependent behaviour.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Python or Node CLI** | Would need to re-implement PSR-4 resolution and a PHP member slicer in a foreign language — more code, more drift, no evidence of benefit. |
| **A language-agnostic core with per-language adapters** | Multi-language support is out of scope by the brief and unsupported by evidence (four PHP commits). It would add an abstraction layer with exactly one implementation. |
| **A Laravel package / Artisan command** | Couples the tool to the codebase under review and drags a framework into a deterministic batch process. The tool must be able to read *any* checkout of the repo, including one it is not installed in. |
| **A hosted service or HTTP API** | No SaaS. Also violates P9 (inputs are free and local) and P8 (no network). |
| **Depend on `nikic/php-parser`, `symfony/console`, etc.** | See [ADR-A004](ADR-A004-tokenizer-slicing.md); a nine-option argv parse and a member slicer do not justify dependencies whose upgrades can change output. |

## Consequences

- The tool runs anywhere PHP 8.2 runs, against any checkout, with `git diff | context-discover`.
- Argv parsing, JSON writing, and member slicing are hand-written against the standard library —
  a deliberate, bounded cost.
- If a future phase must review a non-PHP repository, that is a new architecture phase (new
  `ClassLocator`/`MemberSlicer` semantics), not a configuration flag.
