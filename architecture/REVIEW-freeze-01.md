# Architecture Freeze Review · 01

Independent engineering review of `architecture/` (the implementation contract) against
`AliAlqrinawi/context-discovery-research` @ `main` (the source of truth). Nothing was assumed
correct because it already existed. Findings are ordered by severity; each names the document,
the missing or contradictory evidence, and the smallest correction.

**Verdict: ❌ Architecture Requires Revision.** Five blockers, four over-abstractions, four
leaks/inconsistencies. All corrections are subtractive or one-line additions; none changes the
Phase 1 boundary, and none adds a feature.

---

## A · Blockers — Claude Code cannot implement as written

### B1 · `ItemPriority` is referenced four times and never defined
**Problem.** `01-architecture.md` §3.4 names `Assembly\ItemPriority` as "the fixed, documented
priority order"; `05-traceability.md` (P7 row) and `06-acceptance.md`
(`BudgetEnforcerTest` asserts "drops follow `ItemPriority`") depend on it. **The order is never
stated anywhere.** An implementer must invent it, and the invented order silently decides which
findings survive a tight budget — the one thing P7 exists to make visible.

**Evidence.** The order is derivable, so this is an omission, not a research gap:
`discovery-moves.md` gives cost per move ("effectively free", "low", "higher", "near-zero") and
the frequency table; Exp 1 and 4 make reverse-caller the recurring high-severity move; Exp 2
makes speculative named-reference fetching a precision failure.

**Smallest correction.** Add one table to `01-architecture.md` §3.4, drop-last first:
1 flags (near-zero cost, P10 — never dropped), 2 own-file slices (free, substrate),
3 changed-signature call sites (recurring, high severity), 4 named-reference slices (dropped
first). One sentence stating it is a drop order, not a relevance score (already asserted in
§3.4 — keep that wording).

### B2 · The acceptance fixtures cannot exist
**Problem.** `02-project-structure.md` and `06-acceptance.md` specify
`tests/Acceptance/fixtures/experiment-0N/diff.patch` as project artifacts. The research
repository contains **no diffs** — the four experiment files describe commits in a private
Laravel + Plaid codebase and quote no patch text. The acceptance test — the definition of
"Phase 1 is done" — is therefore unimplementable from the contract, and an implementer will
either fabricate diffs (which would grade the tool against invented ground truth) or silently
skip the test.

**Evidence.** `implementation-spec.md` states the acceptance test is a **by-hand comparison**
against each experiment's minimum-context list. It never claims the diffs are available.

**Smallest correction.** In `06-acceptance.md`: state that the four diffs are an
**operator-supplied prerequisite** exported from the private codebase, that this repository ships
only `expected-context.md` per fixture (transcribed from the experiment files), and that the
harness fails loudly with "fixture diff absent — acceptance not run" rather than passing or
skipping quietly.

### B3 · "Named reference" has no recognition contract
**Problem.** `NamedReferenceAssertionExtractor` is specified as emitting assertions for
"classes/members the region names and that are not defined locally". Which syntactic forms count
is undefined, and that single omission determines R2's entire recall — the requirement carrying
roughly half the context-dependent findings.

**Evidence.** The four forms needed are each pinned by an experiment, so no new evidence is
required: `PlaidAccount` model surface (imported class named in a call — Exp 1);
`upsertFromPlaid` same-class sibling (Exp 1); `PlaidClient::createLinkToken` — a member reached
through an injected collaborator property (Exp 4); `PlaidItemStatus` enum member access (Exp 4).

**Smallest correction.** Add a four-row "reference forms recognised" list to
`01-architecture.md` §3.3, each row citing its experiment, plus one line: any other form is
**not** recognised and produces no assertion (no inference, no guessing) — consistent with
A003's closed set.

### B4 · `filesUnder()` is unordered, which breaks determinism
**Problem.** `Ports\SourceRepository::filesUnder()` (`03-interfaces.md`) has no ordering
contract. Directory iteration order is filesystem- and platform-dependent, so the caller grep
under `--max-call-sites` can return a *different subset* of call sites on two machines. P8
("given the same diff and repository state, the engine produces the same bundle") and the
acceptance test's byte-identical assertion both fail intermittently — the worst failure class,
because it passes locally.

**Smallest correction.** One clause in the port contract: `filesUnder()` returns paths sorted
lexicographically; `CallSiteSearch` returns call sites in that order.

### B5 · `BundleAssembler` takes an untyped tuple standing in for a missing domain type
**Problem.** `03-interfaces.md` specifies
`assemble(array{Assertion, Lever, list<SourceSlice>|string}[] $resolved, int $budgetTokens)`.
A positional tuple with a union third slot is the one place the contract stops being a contract:
an implementer must invent the shape, and `Domain\Bundle`'s "no item without a reason and a
lever" invariant cannot be checked against it.

**Smallest correction.** Add one value type — `Domain\Assertion\ResolvedAssertion`
(assertion, lever, and either slices or a statement) — and type `assemble(list<ResolvedAssertion>,
int)`. Net +1 class, −1 ambiguity; module count 25.

---

## B · Unnecessary abstraction and hidden over-engineering

### O1 · `Ports\DiffParser` is a port with no I/O and one implementation
**Problem.** Ports exist to invert I/O and keep inner layers off the filesystem
(`01-architecture.md` §3.2, P9). `UnifiedDiffParser` performs **no** I/O — it maps text to value
objects — and there is exactly one implementation, with no evidence of a second diff format.
The interface buys nothing and adds an indirection an implementer must wire.

**Smallest correction.** Delete `Ports\DiffParser`; move `UnifiedDiffParser` from `Adapters\Diff`
to `Discovery\Parsing` as a pure final class the pipeline calls directly. Dependency direction
stays inward (Pipeline → Discovery), so no rule is weakened. `Adapters\Diff/` disappears.

### O2 · `Ports\TokenEstimator` is speculative flexibility
**Problem.** Same shape as O1 — pure function, one implementation. Its stated motive
(ADR-A007: "replaced only after measurement") is *future* flexibility, which the project's own
rule — no optimisation, and no infrastructure, before measurement — forbids. Building the seam
now is the cheapest possible over-engineering, but it is still over-engineering.

**Smallest correction.** Delete the port; make it a pure `Assembly\TokenEstimate`.
`Adapters\Estimation/` disappears. Ports: 7 → 5 (`SourceRepository`, `ClassLocator`,
`MemberSlicer`, `CallSiteSearch`, `BundleWriter` — the last justified by two real
implementations selected at runtime).

### O3 · `AssertionResolver::supports()` is a plugin seam in disguise
**Problem.** `03-interfaces.md` defines `supports(Assertion): bool` + `resolve(...)` and the
pipeline loops the resolvers. That is runtime dispatch over a set the architecture insists is
**closed** (ADR-A003). It is exactly the shape a plugin registry grows from, it hides the
closed set from the reader, and it makes "which resolver handled this?" a runtime question
instead of a source-readable fact.

**Smallest correction.** Drop `supports()`. Dispatch explicitly on `AssertionKind` in
`Pipeline\DiscoverContext` (one `match`, four arms, all four kinds visible in one place). Keep
`resolve()` on the interface for uniform typing, or drop the interface too — either is smaller
than today. Add one sentence to ADR-A003 recording why.

### O4 · `--out` is beyond the minimum
**Problem.** `03-interfaces.md` specifies `--out <path>` alongside stdout. Shell redirection
already covers it. "Keep Phase 1 as small as possible" applies to the CLI surface too, and every
option is a branch the acceptance test must consider.

**Smallest correction.** Delete `--out`. The bundle goes to stdout; diagnostics to stderr.

---

## C · Leaked implementation detail and internal inconsistency

### L1 · ADR-A007 fixes a formula and an invented constant — and contradicts ADR-A008
**Problem.** ADR-A007 specifies `ceil(strlen(text) / 4)`. Two defects in one line: it is
implementation code in a repository that forbids implementation code, and the ratio `4` is a
number the research does not contain — while ADR-A008 refuses to ship a `--budget` default
*precisely because* inventing a number hides an untested assumption in the code. The
architecture cannot hold both positions.

**Smallest correction.** Restate A007 as: a single documented characters-per-token ratio,
declared as one named constant, labelled an estimate in the schema; record the ratio's value as
an **Architectural Assumption** (AA4 below) rather than as evidence-derived. Remove the
expression.

### L2 · `diagnostics[]` is duplicated inside the bundle
**Problem.** `01-architecture.md` §5 specifies diagnostics on **stderr**, and
`03-interfaces.md` *also* puts a `diagnostics[]` array in the bundle JSON (and the reviewable
document repeats it). Two channels for the same facts, and it inserts non-item text into the very
artifact whose token cost is the measurement — `used_tokens` counts only items, so the artifact
is now larger than the number that describes it. `implementation-spec.md` fixes the bundle's
contents as items, budget accounting, and the drop list. Diagnostics are not among them.

**Smallest correction.** Remove `diagnostics[]` from the JSON schema (and the sample in the
review document). stderr is already specified, and every diagnostic already has a corresponding
flag item, per A009.

### L3 · `SourceRepository` collides with the architecture's own banned vocabulary
**Problem.** `03-interfaces.md` §5 bans `Repository` "(persistence sense)" while
`02-project-structure.md` lists `SourceRepository`/`LocalSourceRepository` as ports and adapters.
A reader applying the naming rule literally will rename or hesitate.

**Smallest correction.** One clause: "*Repository* is reserved for **the repository under
review**; the persistence-pattern ban stands." No rename.

### L4 · `docs/architecture/` "copied in verbatim" creates a second source of truth
**Problem.** `02-project-structure.md` ships this contract inside the tool repository verbatim.
Two copies drift, and the drift is invisible.

**Smallest correction.** One provenance line at the top of the copy naming its origin and the
review it was frozen at.

---

## D · Architectural Assumptions — reasonable, not proven by the research

Each is defensible and none expands scope, but none is traceable to a Phase 0 finding. They are
listed so the freeze records them as assumptions rather than evidence.

| # | Assumption | Where | Why it is not evidence |
|---|---|---|---|
| AA1 | `--max-call-sites` default `20` | `03-interfaces.md` | A bound is required by P7/P10; the *number* is invented. Exp 4 gives no count |
| AA2 | A Markdown writer exists in addition to JSON (and is what justifies the `BundleWriter` interface) | Adapters, `03-interfaces.md` | Supported only indirectly — ROADMAP calls the output something "a human … pastes alongside the diff". The spec explicitly leaves serialisation open. Dropping Markdown would also drop the interface |
| AA3 | `assertion_kind` on every bundle item | `03-interfaces.md` | R4 requires payload, reason, lever. Machine-readable per-move attribution is an addition; the *reason* field is what the research requires for measurability |
| AA4 | The characters-per-token ratio | ADR-A007 | See L1. Adequate for a coarse comparative claim; unmeasured |
| AA5 | Premises `unresolved-reference` and `call-sites-truncated` | ADR-A009 | Derived from P10, not from any finding. Correct in spirit; no experiment earned them |
| AA6 | PHP 8.2, zero runtime dependencies | ADR-A001 | Strongly *consistent* with the evidence (PSR-4 map input; four PHP commits) but the research never names a language |
| AA7 | The exit-code taxonomy `0/1/2` | `03-interfaces.md` | Ordinary CLI practice; no research basis, none needed |
| AA8 | JSON as the default format and `bundle_version` as a versioned contract | `03-interfaces.md` | The spec says the serialisation is *not* fixed by it |

---

## E · Verified clean

Checked and found correct — recorded so the freeze is not re-litigated:

- **Module → evidence.** All 24 modules appear in `05-traceability.md` with a requirement and an
  experiment. No orphan modules. (After the corrections: 25 classes, 5 interfaces.)
- **Dependency direction.** Rules 1–6 hold as written; `Domain` imports nothing; `Discovery`,
  `Assembly` and `Pipeline` name no adapter; `Cli\Wiring` is the single construction point. O1/O2
  remove two needless inward interfaces without changing a direction.
- **Scope.** X1–X5 are enforced structurally, not by discipline: no forward-following module, no
  worklist type that could hold pending references (depth > 1 is unreachable), no config/migration
  resolver, no severity or comment type, no HTTP client, no dependency that could reach a model.
- **P1 held.** `Domain\Assertion` is the only currency of the middle stages; there is no
  "related files" type anywhere — the reframing survived into the structure.
- **A005 and A008 are the strongest decisions in the set.** A005 is what keeps Experiment 2's
  bundle almost empty; A008 is the one place the architecture refuses to invent a number the
  research does not contain. L1 must be fixed to match A008's own standard.
- **Interface responsibilities.** Each of the five surviving ports has one job.
  `MemberSlicer`'s four methods are one responsibility (PHP text → member spans) and all four are
  consumed — `memberNames()` by the extractor's "not defined locally" test.

---

## F · Revision list (all subtractive or one-line)

1. State the `ItemPriority` order (B1).
2. Declare acceptance fixtures operator-supplied; harness fails loudly when absent (B2).
3. Add the four recognised reference forms, each cited (B3).
4. Require lexicographic ordering from `filesUnder()`/`CallSiteSearch` (B4).
5. Add `ResolvedAssertion`; retype `assemble()` (B5).
6. Delete `Ports\DiffParser`; move the parser to `Discovery\Parsing` (O1).
7. Delete `Ports\TokenEstimator`; pure `Assembly\TokenEstimate` (O2).
8. Drop `supports()`; explicit `match` on `AssertionKind` (O3).
9. Delete `--out` (O4).
10. Remove the formula from A007; record the ratio as AA4 (L1).
11. Remove `diagnostics[]` from the bundle schema (L2).
12. Clarify the *Repository* naming rule (L3).
13. Add a provenance line to the shipped copy of the contract (L4).
14. Record AA1–AA8 in `evidence-gaps.md` under a new "Architectural assumptions" heading.

Net effect: 7 ports → 5, one CLI option removed, one schema field removed, one value type added,
four omissions closed. No module gained, no capability added, no boundary moved.

**❌ Architecture Requires Revision**
