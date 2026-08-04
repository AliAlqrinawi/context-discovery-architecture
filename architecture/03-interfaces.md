# 03 · Interfaces

Two surfaces. The **public** one is the CLI contract plus the bundle schema — that is what Phase 2
scoring and any human consumer depend on, and it is versioned. The **internal** one is the five
ports and the discovery collaborators; nothing outside the tool depends on them, so they are free
to change.

Signatures are given as declarations only. There is no implementation code in this repository.

---

## 1 · Public interface · the command

```
context-discover --diff <path|-> --repo <path> --budget <int>
                 [--format json|markdown]
                 [--caller-scope <prefix>] [--max-call-sites <int>]
```

| Option | Required | Default | Meaning |
|---|---|---|---|
| `--diff` | yes | — | Unified diff for the commit under review; `-` reads stdin |
| `--repo` | yes | — | Repository root. All reads are scoped inside it |
| `--budget` | **yes** | none | Token budget. No default: the research fixes no number ([ADR-A008](decisions/ADR-A008-required-budget.md)) |
| `--format` | no | `json` | `json` (stable machine contract) or `markdown` (paste alongside the diff) |
| `--caller-scope` | no | `app/` | Path prefix for the reverse-caller grep (Exp 4's minimum context) |
| `--max-call-sites` | no | `20` | Hard bound on call sites per changed signature; the excess is recorded as a **flag**, never dropped silently (P10). The bound's *value* is an architectural assumption (AA1) |

**Contract**

- Deterministic: same diff + same repository state ⇒ byte-identical output (P8).
- Read-only: the tool never writes inside `--repo`.
- No network, no subprocess, no LLM, no state directory (P9, X5).
- Diagnostics go to stderr only; the bundle on stdout is always parseable and carries no diagnostics.
- Exit codes: `0` bundle produced (including an empty bundle) · `1` usage or input error ·
  `2` repository unreadable.

## 2 · Public interface · the bundle (JSON, `bundle_version: 1`)

```json
{
  "bundle_version": 1,
  "budget_tokens": 8000,
  "used_tokens": 2143,
  "items": [
    {
      "lever": "fetched",
      "reason": "changed call site depends on PlaidAccount::forItem() and official_name",
      "assertion_kind": "named_reference",
      "provenance": {
        "path": "app/Models/PlaidAccount.php",
        "member": "forItem",
        "lines": [41, 58]
      },
      "payload": "<minimal slice>",
      "tokens": 180
    },
    {
      "lever": "flagged",
      "reason": "reconciliation assumes a surrounding DB transaction; caller not verified",
      "assertion_kind": "unverifiable_premise",
      "provenance": {
        "path": "app/Services/Plaid/PlaidAccountService.php",
        "member": "syncFromResponse",
        "lines": [120, 168]
      },
      "payload": "ASSUMPTION: this code assumes a surrounding transaction; caller not checked",
      "tokens": 14
    }
  ],
  "dropped": [
    { "reason": "route wiring for reauth endpoint", "note": "below budget priority", "tokens": 260 }
  ]
}
```

**Field rules**

| Field | Rule |
|---|---|
| `bundle_version` | Integer. Bumped on any breaking change to this schema |
| `budget_tokens` | Echoes `--budget` |
| `used_tokens` | Sum of `items[].tokens`; always ≤ `budget_tokens` |
| `items[].lever` | `fetched` \| `flagged`. Required (P5) |
| `items[].reason` | Non-empty string naming the reference or assumption resolved. Required — an item without one is a defect, not a warning (P5) |
| `items[].assertion_kind` | `same_file_symbol_absence` \| `named_reference` \| `changed_signature` \| `unverifiable_premise`. Present so precision can be measured **per move** after the scored run |
| `items[].provenance` | `path`, optional `member`, optional `lines` — enough for a human to verify the slice by hand |
| `items[].payload` | Fetched: the minimal source slice. Flagged: the assumption sentence, nothing else |
| `dropped[]` | Every budget drop, with its reason. Empty array when nothing was dropped. Never omitted (P7) |

Unresolved references, unreadable paths and missing PSR-4 entries go to **stderr** only. Each also
produces a flag item (ADR-A009), so nothing in the bundle depends on the diagnostic stream.

**Ordering** (fixed, so output is diffable): items sorted by `assertion_kind` in the order
`same_file_symbol_absence`, `changed_signature`, `named_reference`, `unverifiable_premise`, then by
`provenance.path`, then by `provenance.member`. Drops keep the order in which they were dropped.

**Markdown format** — same information, human-readable, one `##` heading per item carrying lever,
reason, and provenance, followed by a fenced payload; a final `## Dropped` section and a
`budget / used` line. It is a projection of the JSON, never a different set of facts.

## 3 · Internal interfaces · ports

```php
interface SourceRepository {                       // root-scoped, read-only
    public function exists(string $relativePath): bool;
    public function text(string $relativePath): ?string;
    /** @return list<string> sorted lexicographically — required by P8 */
    public function filesUnder(string $prefix, string $extension): array;
}

interface ClassLocator {                           // PSR-4 map, depth-one, no index
    public function pathFor(string $fullyQualifiedClass): ?string;
}

interface MemberSlicer {                           // PHP text → minimal slices
    public function useBlock(string $fileText): ?SourceSlice;
    public function member(string $fileText, string $memberName): ?SourceSlice;
    /** @return list<string> */
    public function memberNames(string $fileText): array;
    public function enclosingMemberName(string $fileText, int $line): ?string;
}

interface CallSiteSearch {                         // a bounded grep, never a graph
    /** @return list<CallSite> in filesUnder() order, then by line — required by P8 */
    public function callSites(string $methodName, string $scopePrefix, int $max): array;
}

interface BundleWriter {
    public function write(Bundle $bundle): string;
}
```

## 4 · Internal interfaces · discovery collaborators

```php
final class AssertionExtractor {
    /** @return list<Assertion> */
    public function extract(Diff $diff, SourceRepository $source): array;
}

interface RegionAssertionExtractor {               // implemented by the four extractors only
    /** @return list<Assertion> */
    public function forRegion(ChangedFile $file, ChangedRegion $region, string $fileText): array;
}

final class LeverPolicy {
    public function leverFor(Assertion $assertion, ClassLocator $locator): Lever;
}

interface AssertionResolver {                     // OwnFile / NamedReference / Caller
    /** @return list<SourceSlice> empty list ⇒ caller must flag instead (P10) */
    public function resolve(Assertion $assertion): array;
}

final class AssumptionWriter {
    public function statementFor(Assertion $assertion): string;
}

final class UnifiedDiffParser {                   // pure; not a port
    public function parse(string $diffText): Diff;
}

final class TokenEstimate {                       // pure; not a port
    public function of(string $text): int;
}

final class BundleAssembler {
    /** @param list<ResolvedAssertion> $resolved */
    public function assemble(array $resolved, int $budgetTokens): Bundle;
}

final class BudgetEnforcer {
    public function enforce(Bundle $bundle): Bundle;   // returns a bundle with drops recorded
}
```

`RegionAssertionExtractor` and `AssertionResolver` exist **only** to keep the four extractors and
three resolvers uniform inside the pipeline. They are not extension points: implementations are a
closed set, constructed explicitly in `Cli\Wiring`, with no discovery, registration, or
configuration ([ADR-A003](decisions/ADR-A003-closed-move-set.md)). Dispatch is an explicit `match` on
`AssertionKind` inside `Pipeline\DiscoverContext` — there is no `supports()` probe — so all four kinds
and their resolvers are visible in one place.

## 5 · What deliberately has no interface

- No `Reviewer`, `Prompt`, or `LlmClient` (X5, P6).
- No `Severity`, `Score`, or `Ranking` (X4).
- No `ConfigResolver` / `MigrationResolver` — config and schema go through the flag path (X3).
- No `ImportFollower` / `DependencyGraph` (X1, X2, P4).
- No `Cache`, `Index`, `Repository` (persistence sense), or `EventDispatcher`. *Repository* is reserved
  for the repository under review — that is what `SourceRepository` names.
- No `DiffParser` or `TokenEstimator` port: both are pure text transformations with one
  implementation, so they are plain classes (freeze review O1/O2).
