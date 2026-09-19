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
                 [--format json|markdown] [--repo-sha <sha>]
                 [--caller-scope <prefix>] [--max-call-sites <int>]
```

| Option | Required | Default | Meaning |
|---|---|---|---|
| `--diff` | yes | — | Unified diff for the commit under review; `-` reads stdin |
| `--repo` | yes | — | Repository root, holding the tree **as `--diff` leaves it** — the post-image. All reads are scoped inside it |
| `--budget` | **yes** | none | Token budget. No default: the research fixes no number ([ADR-A008](decisions/ADR-A008-required-budget.md)) |
| `--format` | no | `json` | `json` (stable machine contract) or `markdown` (paste alongside the diff) |
| `--caller-scope` | no | `app/` | Path prefix for the reverse-caller grep (Exp 4's minimum context) |
| `--max-call-sites` | no | `20` | Hard bound on call sites per changed signature; the excess is recorded as a **flag**, never dropped silently (P10). The bound's *value* is an architectural assumption (AA1) |
| `--repo-sha` | no | none | Recorded verbatim in `run.repo_sha`; `null` when not given. Supplied, never read: `git rev-parse` needs a subprocess (P9), and `.git` is a *file* in the detached worktree ADR-A022's harness uses ([ADR-A024](decisions/ADR-A024-bundle-contract-v2.md)) |

**Contract**

- Deterministic: same diff + same repository state ⇒ byte-identical output (P8).
- **`--repo` is the post-image**: the tree with `--diff` already applied, so a line the diff adds
  is a line the tree contains. Extraction reads post-image line numbers against these files
  ([ADR-A023](decisions/ADR-A023-changed-return-contract.md) is the first move to depend on it;
  the harness of [ADR-A022](decisions/ADR-A022-experiments-resolve-at-the-reviewed-commit.md)
  satisfies it by checking the reviewed commit out into a detached worktree). **A pre-image tree
  is not merely a smaller bundle: it can be a wrong one.** `OwnFileAssertionExtractor` reads the
  file's `use` block, so an import the diff adds is not seen and a spurious
  `same_file_symbol_absence` is claimed; the premise and own-file resolvers scan post-image line
  numbers and can name the wrong member. The tool therefore checks the contract as a count: when
  **none** of the non-blank lines the diff adds is present in any changed file, it emits
  `post-image contract violated` on stderr and a `post_image_contract_violated` entry in
  `diagnostics[]`. Only that categorical case is stated — a partial count would be a judgement
  (P6) — and it is a diagnostic, never a flag, because it is an input violation rather than a
  lookup failing (ADR-A023).
- Read-only: the tool never writes inside `--repo`.
- No network, no subprocess, no LLM, no state directory (P9, X5).
- Diagnostics go to stderr only; the bundle on stdout is always parseable and carries no diagnostics.
- A successful negative result is a diagnostic, not an item and not a flag (freeze review 05).
- Exit codes: `0` bundle produced (including an empty bundle) · `1` usage or input error ·
  `2` repository unreadable.

## 2 · Public interface · the bundle (JSON, `bundle_version: 2`)

Frozen by **`schema/bundle-v2.schema.json`**, which is the single source: the closed sets below are
asserted against the code by `BundleSchemaConformanceTest`, so a kind, lever or diagnostic type
added in one place and not the other fails the build. This section describes the schema; the schema
decides ([ADR-A024](decisions/ADR-A024-bundle-contract-v2.md)).

The bundle states each **claim once** and has the evidence point back at it. v1 repeated one reason
into every item — twenty-one copies of one sentence on the M26 reproduction — and left the claim's
subject recoverable only by parsing prose.

```json
{
  "bundle_version": 2,
  "run": {
    "engine_version": "0.2.0",
    "policy_version": "2",
    "framework_table_version": "5bfb119e1a00",
    "budget_tokens": 8000,
    "diff_sha": "6070b2d17713",
    "repo_sha": null
  },
  "used_tokens": 2143,
  "assertions": [
    {
      "id": "a1f4c0b39e77a",
      "kind": "named_reference",
      "subject": "App\\Models\\PlaidAccount::forItem",
      "reason": "changed call site depends on PlaidAccount::forItem() and official_name",
      "origin": { "path": "app/Services/Plaid/PlaidAccountService.php", "lines": [120, 168] }
    }
  ],
  "items": [
    {
      "assertion_id": "a1f4c0b39e77a",
      "lever": "fetched",
      "provenance": {
        "path": "app/Models/PlaidAccount.php",
        "member": "forItem",
        "lines": [41, 58]
      },
      "payload": "<minimal slice>",
      "tokens": 180
    }
  ],
  "diagnostics": [
    {
      "type": "call_sites_truncated",
      "assertion_id": "a1f4c0b39e77a",
      "detail": { "limit": 20, "subject": "forItem", "scope": "app/" }
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
| `bundle_version` | Integer, now `2`. Bumped on a **removal, rename, re-nesting, type change or change of meaning**; an added field, kind or diagnostic type is additive and does not bump it. The full policy is ADR-A024's table. *(v1's justification — "no bundle has been emitted by a working tool" — is retired: around ninety v1 bundles are committed under `tests/Acceptance/fixtures/experiment-*`. They are frozen at v1, never regenerated and never translated; comparing across the boundary means re-running.)* |
| `run` | What produced the bundle. Nothing here is observed from the environment: no clock, no hostname, no process (P8, P9) |
| `run.engine_version` | Declared constant, bumped with the release |
| `run.policy_version` | Declared constant covering `LeverPolicy`, `ItemPriority` and `PremiseCatalogue`. **Guarded**: a test hashes those classes and fails when they move without it moving |
| `run.framework_table_version` | **Derived** from the framework naming table's own content — it is pure data, so a content hash is honest |
| `run.budget_tokens` | Echoes `--budget`. Moved here from the root in v2 |
| `run.diff_sha` | Digest of the diff bytes handed in |
| `run.repo_sha` | From `--repo-sha`, or `null` — emitted explicitly, so "not supplied" is stated rather than inferred |
| `used_tokens` | Sum of `items[].tokens`, **and of nothing else**. Diagnostics are excluded by design (freeze review L2) |
| `assertions[].id` | Stable, derived from kind, subject, origin path and first line — the tuple extraction already de-duplicates on (P8). No counter |
| `assertions[].kind` | `same_file_symbol_absence` \| `same_file_reference` \| `named_reference` \| `changed_signature` \| `changed_return_contract` \| `unverifiable_premise`. One value per discovery move, so precision can be measured **per move** after the scored run — and so `ItemPriority` can band an item from this field alone |
| `assertions[].subject` | The extractor's own structured datum, never parsed from the reason. **Polymorphic by kind**: a member name for the signature and return-contract kinds; a fully-qualified class or member for a named reference; a symbol for the same-file kinds; a catalogue identifier for a premise |
| `assertions[].reason` | Non-empty. Required — an assertion without one is a defect, not a warning (P5) |
| `assertions[].origin` | `path` and an optional line span. **No member**: an assertion has no origin member, so none is claimed |
| `items[].assertion_id` | Must match an `assertions[].id`. Enforced in the domain: an orphan item is rejected before serialisation, which is how P5 survives the reason moving (ADR-A024) |
| `items[].lever` | `fetched` \| `flagged`. Required (P5) |
| `items[].provenance` | `path`, optional `member`, optional `lines` — enough for a human to verify the slice by hand |
| `items[].provenance.member` | **Optional, and absent rather than null.** Present when the slice *is* a member: the enclosing member, a named reference's declaration, a model or enum surface, and a flagged item whose assertion names one ([ADR-A009](decisions/ADR-A009-premise-catalogue.md)). Absent for a file's `use` block, which has no member name; for a flagged premise, which names a premise; and for **every reverse-caller call site**, because a call site is a line, not a member |
| `items[].payload` | Fetched: the minimal source slice. Flagged: the assumption sentence, nothing else |
| `diagnostics[]` | A machine-readable **mirror** of stderr, never a relocation — stderr still carries every line byte for byte, because a harness uses it to tell "searched, found none" from "never searched" (freeze review 05). Empty rather than absent. **Costs no tokens** |
| `diagnostics[].type` | Closed set, currently `call_sites_truncated`. Mirroring a second diagnostic is additive and requires editing the schema |
| `dropped[]` | Every budget drop, with the reason its assertion states. Empty array when nothing was dropped. Never omitted (P7) |

Unresolved references, unreadable paths and missing PSR-4 entries go to **stderr**. Each also
produces a flag item (ADR-A009), so nothing in the bundle depends on the diagnostic stream.

One diagnostic class has **no** corresponding item: a successful negative result — a caller search
that completes with zero call sites, or a changed file with no `use` block. It is recorded on stderr
(`caller search for reactivate( under app/: 0 call sites`) so the acceptance harness can tell "searched,
found none" from "never searched"; it produces no bundle item and no flag (freeze review 05).

The `lever: flagged` ASSUMPTION item is **retained** beside `diagnostics[]` rather than replaced by
it. They serve different readers — the flag is context an agent acts on inside the bundle, and it is
protected from budget drops (P10); the mirror is for a consumer that wants values. The cost is
stated: 21 tokens on the M26 reproduction (ADR-A024).

**Ordering** (fixed, so output is diffable): items sorted by their assertion's `kind` in the order
`same_file_symbol_absence`, `same_file_reference`, `changed_signature`, `changed_return_contract`,
`named_reference`, `unverifiable_premise`, then by
`provenance.path`, then by `provenance.member`. Drops keep the order in which they were dropped.
The order lives in the schema at `$defs/kindOrder`; `BundleAssembler::KIND_ORDER` is a copy the
conformance test holds to it.

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
    public function enclosingMemberName(string $fileText, int $line): ?string;   // which member am I in?
    public function memberOwningLine(string $fileText, int $line): ?string;      // whose statement is this?
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

interface RegionAssertionExtractor {               // implemented by the five extractors only
    /** @return list<Assertion> */
    public function forRegion(ChangedFile $file, ChangedRegion $region, string $fileText): array;
}

final class LeverPolicy {
    public function leverFor(Assertion $assertion, ClassLocator $locator): Lever;
}

// Resolver dispatch is an explicit match on AssertionKind in Pipeline\DiscoverContext:
//   SameFileSymbolAbsence, SameFileReference → OwnFileResolver
//   NamedReference                          → NamedReferenceResolver
//   ChangedSignature, ChangedReturnContract  → CallerResolver
//   UnverifiablePremise                     → AssumptionWriter (flag; never resolved)
// Every kind has exactly one destination; no probing, no fall-through.

interface AssertionResolver {                     // OwnFile / NamedReference / Caller
    /**
     * @return list<SourceSlice> An empty list is one of TWO cases, distinguished by the
     *   resolver, never by the pipeline:
     *   (a) lookup FAILURE — the lookup could not be performed or the source that should
     *       exist could not be read (no PSR-4 entry, unreadable path, member not found;
     *       unreadable caller scope). Becomes a flag, using the premise belonging to THAT
     *       resolver — `unresolved-reference` for a named reference, `caller-search-failed`
     *       for a caller search. Premises are never shared across resolvers (P10,
     *       freeze review 06).
     *   (b) successful NEGATIVE — the question was answered and the answer is "none"
     *       (a caller search that completes with zero call sites; a changed file with no
     *       `use` block). No premise exists to state, so: no bundle item, one stderr
     *       diagnostic. Emitting a flag here would assert an assumption that is not being
     *       made — a precision failure (freeze review 05).
     */
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

`RegionAssertionExtractor` and `AssertionResolver` exist **only** to keep the five extractors and
three resolvers uniform inside the pipeline. They are not extension points: implementations are a
closed set, constructed explicitly in `Cli\Wiring`, with no discovery, registration, or
configuration ([ADR-A003](decisions/ADR-A003-closed-move-set.md)). Dispatch is an explicit `match` on
`AssertionKind` inside `Pipeline\DiscoverContext` — there is no `supports()` probe — so all six kinds
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
