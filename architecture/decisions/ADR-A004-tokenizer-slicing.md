# ADR-A004 · Slice members with PHP's built-in tokenizer — no AST library, no regex

- **Status:** Accepted
- **Date:** 2026-08-05
- **Phase:** Phase 1 architecture
- **Implements:** R1, R2, P9, ADR-A001

## Why this decision exists

Both fetch-side requirements demand *minimal slices*: "the smallest slice that settles the
assertion — ideally one method or class member, not a whole file." Producing that slice requires
knowing where a member starts and ends, where the `use` block is, and which member encloses a
changed line. Something must decide how much parsing machinery that justifies.

## Decision

`Adapters\Php\TokenizerMemberSlicer` uses PHP's built-in tokenizer (`token_get_all`) to find:

- the `use` block span,
- the span of a named class member (signature + body, by brace balance),
- the list of member names in a file,
- the member enclosing a given line.

No AST library. No regular-expression parsing of PHP structure. No caching of parse results.

## Evidence from Phase 0

- Minimum-context lists are member-level, not file-level: "the `use` block plus the full
  `syncFromResponse` body", "`upsertFromPlaid` method from the same repository file",
  "`PlaidClient::createLinkToken` — one method (the mode switch)", "`PlaidItemStatus` — one small
  class" (Exp 1, 4). A file-level fetch would fail the precision criterion in the spec.
- Exp 1's missing-`Log`-import finding requires reading the `use` block as a structure — a
  brace-and-token concern, not a text search.
- P9 forbids maintained indexes; a tokenizer pass per file, per run, satisfies it.
- P8 demands identical output for identical input: the tokenizer is part of the language runtime
  pinned by ADR-A001.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **`nikic/php-parser` (full AST)** | Adds a runtime dependency (against ADR-A001) whose version can change slice boundaries between environments, for capability the tool does not need: no type inference, no name resolution beyond PSR-4, no traversal (P3). |
| **Regular expressions over source text** | Cannot balance braces or distinguish `use` imports from closure `use` clauses reliably; would produce wrong slices, which corrupts precision measurement. |
| **Fetch whole files instead of slices** | Directly contradicts the spec's "not a whole file", inflates tokens against P7, and would turn Experiment 2's correct near-empty bundle into a precision failure. See [ADR-A005](ADR-A005-slices-not-files.md). |
| **Build a symbol index for fast lookup** | P9; and no measurement has shown lookup to be a problem (no optimisation before measurement). |

## Consequences

- Slice quality is bounded by tokenizer-level understanding: the slicer finds *named* members, and
  cannot resolve dynamic or aliased names. When it cannot find a member, the assertion is
  **flagged**, not silently skipped (P10).
- Attribute/annotation lines immediately preceding a member are included in its slice, since they
  are part of what a reviewer must see.
