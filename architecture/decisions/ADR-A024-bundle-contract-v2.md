# ADR-A024 · The bundle states its claims once, and says what produced it

- **Status:** Accepted
- **Date:** 2026-09-09
- **Phase:** Phase 1 · freezes the published output contract at `bundle_version` 2
- **Implements:** R4, P5, P6, P7, P8, P9, P10, ADR-A005, ADR-A021
- **Adds:** `assertions[]`, `diagnostics[]` and `run` to the bundle; a versioned JSON Schema and a
  conformance test; one CLI option, `--repo-sha`. **No change to what is discovered** — not one
  extractor, resolver, lever, band or budget rule moved.

## Why this decision exists

The v1 bundle repeated itself and hid what it knew.

The M26 reproduction is the clearest case: one changed member, one claim, **twenty-one items**, and
the same 165-character sentence copied into every one of them. The thing being justified was the
claim; the items were evidence for it. Worse, the claim's *subject* — `getAll`, the single most
useful fact in the bundle — existed nowhere as data. A consumer wanting it had to regex it out of
English prose. That is not a contract; it is a report that happens to be machine-readable.

Three smaller failures pointed the same way:

- `KIND_ORDER`, the fixed item ordering, lived in `BundleAssembler`, in `03-interfaces.md` and in
  `ExperimentKeyTest` with **nothing forcing agreement**. It went stale in the third for a whole
  commit and no test noticed.
- The `bundle_version` field carried a rule whose justification had quietly become false.
- `provenance.member` was present on some items and absent on others with no written rule, so a
  consumer could only learn the pattern by reading the assembler.

## Decision

> **The bundle states each claim once, in `assertions[]`, and items reference it by
> `assertion_id`. It reports what produced it in `run`. It mirrors its diagnostics as data. The
> shape is frozen by `schema/bundle-v2.schema.json`, which is the single source, and a conformance
> test fails the build when the code and the schema disagree.**

Nothing about discovery changed. The M26 reproduction emits the **same 22 items in the same order
for the same 562 tokens**, every visible field identical, and stderr byte for byte unchanged. That
was the constraint the work was done under, and it was verified field by field rather than asserted.

### The reason moved, and P5 moved with it

`BundleItem` no longer carries `reason` or `assertion_kind`. **This is a contract change, not a
rename**, and the P5 invariant — *nothing is shown to a reviewer without a stated reason* — is now
enforced in two places instead of one:

1. `BundleAssertion` rejects an empty reason, exactly as `BundleItem` used to.
2. `Bundle` rejects an **orphan item**: one naming an assertion the bundle does not carry.

The second guard exists because v2 bought de-duplicated output at the price of a reference that
could dangle. An item pointing at nothing is unjustified context — precisely the defect the old
per-item field prevented — so the reference is checked on construction, before anything is
serialised. The guard is one-directional by design: an assertion with no items is fine, because
ADR-A021 collapses indistinguishable evidence and the budget drops the rest.

### `subject` is the extractor's own datum

`Assertion->subject` has always been structured data, set by each extractor from what it actually
found; assembly simply discarded it. v2 publishes it. Nothing is parsed out of prose.

It is **polymorphic by kind**, and the schema says so rather than pretending otherwise: a member
name for `changed_signature` and `changed_return_contract`; a fully-qualified class or member for
`named_reference`; a symbol for the same-file kinds; a catalogue identifier for
`unverifiable_premise`.

`assertion.origin` is a path and a line span and **nothing more**. An assertion has no origin
member — the concept does not exist upstream — so none is invented.

### Diagnostics are mirrored, never moved

Freeze review L2 kept diagnostics out of the bundle because they would make the artifact larger than
the `used_tokens` number describing it. Freeze review 05 made stderr load-bearing in a second way:
it is how a harness tells *"searched, found none"* from *"never searched"*, and
`baseline-v0.1.0.json` records the exact lines per scenario.

Both objections are answered rather than overruled:

- **stderr still carries every line, byte for byte.** `diagnostics[]` is an added copy for machines,
  not a relocation. Moving them would have broken the harness and fourteen recorded baselines.
- **Diagnostics carry no tokens and are excluded from `used_tokens`,** which still sums the items
  and nothing else. L2's objection was about the number, and the number is unaffected.

The existing `lever: flagged` ASSUMPTION item **stays**, and this is a decision rather than an
oversight. The two serve different readers: the flag is context an agent reads inside the bundle it
was given, and it is protected from budget drops because a silent omission is indistinguishable from
"nothing needed" (P10). `diagnostics[]` is for a consumer that wants values instead of a sentence.
The cost is stated plainly: **21 tokens of budget** on the M26 reproduction, spent on the
`call-sites-truncated` flag. That is the price of the flag remaining legible to the reader who
actually acts on it.

The diagnostic `type` enum is closed. Mirroring a second diagnostic is additive and requires editing
the schema — that edit is the point.

### The three versions, and what each is worth

`run` reports what produced the bundle so two scored runs can be compared knowingly. Nothing in it is
observed from the environment: no clock, no hostname, no process (P8, P9).

| Field | Source | Why that source |
|---|---|---|
| `engine_version` | Declared constant | The tool spawns no process, so a git tag is unreadable from inside it, and `composer.json` carries no version. A hand-maintained constant is the only source that is both in-process and true |
| `policy_version` | Declared constant, **hash-guarded** | It claims to describe `LeverPolicy`, `ItemPriority` and `PremiseCatalogue`. `PolicyVersionGuardTest` hashes those three with comments stripped and fails when they move without the constant moving. A constant nobody is forced to bump would answer the comparison question wrongly and silently, which is worse than not answering it |
| `framework_table_version` | **Genuinely derived** | The cardinality, facade and scope rules are pure data, so a content hash is honest and stable under a prose edit. No constant to forget |

`repo_sha` is **supplied by the caller** through `--repo-sha`, and is `null` when it was not given —
emitted explicitly rather than omitted, so "not supplied" is a stated fact. The tool cannot read it:
`git rev-parse` needs a subprocess, which P9 forbids and `ArchitectureBoundaryTest` enforces, and
reading `.git/HEAD` fails for exactly the case that matters, since ADR-A022's harness checks the
reviewed commit out as a **detached worktree** where `.git` is a file. `diff_sha` is a hash of the
input bytes and needs nothing.

### `provenance.member` is optional, and the schema now says when

Present when the slice **is** a member: the enclosing member, a named reference's declaration, a
model or enum surface, and a flagged item whose assertion names one. Absent for a file's `use`
block, which has no member name; for a flagged premise, which names a premise rather than a member
(ADR-A009); and for **every reverse-caller call site**, because a call site is a line and not a
member. Absent rather than null, as in v1.

### The schema is the source, and a test enforces it

`KIND_ORDER` is why. The closed sets are asserted **both ways** — every value the code can emit is
in the schema, and every value the schema permits exists in the code — plus the fixed ordering, and
the version constant. Removing one kind from the schema fails four separate assertions, which was
verified by doing it.

Validation is a purpose-built walker rather than a library: the runtime has zero dependencies
(ADR-A001) and the subset of JSON Schema this contract uses is narrower than a dependency is worth.

## The v1 artefacts are frozen, and are never translated

Roughly **ninety v1 bundles** are committed under `tests/Acceptance/fixtures/experiment-05` through
`experiment-25`. They are the recorded outputs of M17 through M25 — the scored runs, the coverage
distribution, the defect corpus, the independent replication.

> **They stay at v1. They are not regenerated, not migrated, and not translated.**

No test reads them; they are evidence, not fixtures. A recorded bundle is what a reviewer actually
saw when they answered a scored question, and rewriting it into a newer shape would silently change
the artifact a published measurement rests on.

**Any comparison across the version boundary requires re-running under v2, never translating.** A
translated v1 bundle would carry v2's structure with v1's discovery behind it, and every difference
would then be unattributable — shape or substance, no way to tell. Phase 6 depends on this being
written down before someone reaches for a converter.

## Retiring a sentence that had become false

`03-interfaces.md` §2 said `bundle_version` stays 1 because *"no bundle has been emitted by a working
tool, so v1 was never published to break (freeze review 04)."*

That was true when written and is **false now**: the tool has emitted around ninety bundles, and they
are committed. The rule survives on its own merits — additive changes do not bump the version — but
its justification does not, and a rule resting on a false premise is one nobody can apply. The
sentence is replaced with what is actually true: v1 artefacts exist, they are frozen, and the bump
to 2 is what a restructuring requires.

## The compatibility policy

| Change | Version |
|---|---|
| Adding an optional field to an object | **Additive.** No bump |
| Adding a value to `assertion_kind` — as ADR-A023 did | **Additive.** No bump; the schema's enum is edited and the conformance test enforces the edit |
| Adding a value to the `diagnostic.type` enum | **Additive.** No bump |
| Adding an element to `assertions[]`, `items[]`, `diagnostics[]` or `dropped[]` | **Additive.** These are collections; their contents are output, not contract |
| Making a required field optional | **Additive** for a reader, so no bump — but it needs its own ADR, because it weakens a guarantee |
| **Removing** a field, or renaming one | **Bump** |
| **Re-nesting** a field — `budget_tokens` moving into `run`, as here | **Bump** |
| Changing a field's type, including making it nullable | **Bump** |
| Changing the fixed item ordering | **Bump.** Output is meant to be diffable across runs |
| Changing what a field *means* while keeping its name and type | **Bump**, and the loudest one, because no consumer can detect it |

The last row is the one this ADR most wants remembered. A silent change of meaning is the failure a
version number exists to prevent, and it is the only kind a schema validator cannot catch.

## Consequences

**Positive**

- The M26 bundle states its one claim once instead of twenty-one times, and publishes `getAll` as a
  field instead of burying it in a sentence.
- The ordering has one source. The `KIND_ORDER` class of bug cannot recur silently.
- Two runs can be compared knowing whether the engine, the policy or the framework table moved.
- A golden fixture makes the next shape change a reviewable diff.

**Negative, stated plainly**

- The bundle is a **join**, not a flat list. A consumer that wants an item's kind now looks it up.
  That is the price of not repeating the reason, and it is paid by every reader.
- **Thirteen test files changed** for a release that discovers nothing new. Shape churn is real work
  and buys no recall.
- `policy_version` is only as honest as the guard around it. If someone bumps the hash without
  bumping the constant, the field lies and nothing catches it. The guard makes the right move easy,
  not mandatory.
- `repo_sha` is usually `null`, because it depends on a caller that knows to pass it.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Keep `reason` on the item as well, for compatibility** | Then nothing is fixed: the duplication that motivated v2 stays, and two sources of one fact can drift |
| **Move diagnostics off stderr entirely** | Breaks freeze review 05's harness contract and fourteen recorded stderr baselines. Mirroring costs nothing and breaks neither |
| **Count diagnostics in `used_tokens`** | Freeze review L2's objection exactly: the artifact would outgrow the number describing it |
| **Read `repo_sha` from `.git/HEAD`** | Needs git-internals knowledge in a tool that has none, and returns nothing for the detached worktree ADR-A022's own harness produces |
| **Derive `engine_version` from a git tag** | Requires a subprocess. P9 forbids it and a boundary test enforces the ban |
| **A JSON Schema library** | The runtime is zero-dependency by ADR-A001, and this contract uses a small enough subset that a validator is cheaper to read than a dependency is to justify |
| **Translate the v1 fixtures to v2** | Would put v2's shape around v1's behaviour and make every later difference unattributable. See above |
| **Drop the flagged ASSUMPTION item now that `diagnostics[]` exists** | They serve different readers, and the flag is the one an agent acts on. Its 21-token cost is recorded rather than quietly reclaimed |

## Related

- [ADR-A021](ADR-A021-item-identity-is-the-whole-visible-item.md) — item identity, still computed
  from what a reviewer sees and not from the new `assertion_id`, so v2 collapses exactly what v1
  collapsed.
- [ADR-A023](ADR-A023-changed-return-contract.md) — the move whose 21-fold duplication made the case.
- [ADR-A022](ADR-A022-experiments-resolve-at-the-reviewed-commit.md) — the detached worktree that
  makes `repo_sha` unreadable from inside the tool.
- [ADR-A009](ADR-A009-premise-catalogue.md) — why a flagged premise carries no member.
- [ADR-A001](ADR-A001-php-cli-zero-dependencies.md) — why the validator is hand-written.
- `schema/bundle-v2.schema.json`, `tests/Acceptance/BundleSchemaConformanceTest.php`,
  `tests/Acceptance/fixtures/golden/m26-bundle.v2.json`.

---

## Addendum (2026-09-26) — a templated flag payload is additive; `bundle_version` stays 2

[ADR-A028](ADR-A028-inherited-member-statement-gate-step.md) §7 recorded the argument and left the
decision here. Option B was built on the additive reading (engine `6a77cdb`, 2026-09-24) before
this was written; that order was wrong — the gate should have been cleared before the constant was
relied on — and is corrected by recording the decision now rather than by rewriting the commit.

**Decision.** S1's `items[].payload` — the `inherited-member-declared` statement, rendered from
ADR-A028 §5's template — is an **additive** change under the table above. `bundle_version` stays
**2**.

**Why it is not the last row.** The row that bumps is *"changing what a field means while keeping
its name and type"*. `items[].payload` for `lever: flagged` means, per `03-interfaces.md`, *"the
assumption sentence, nothing else"*. It still does: one sentence, beginning `ASSUMPTION:`, stating
one premise a file could not settle, and nothing else — no slice, no second sentence, no structured
sub-fields. A reader that treated the payload as an opaque sentence before treats it the same way
now. What changed is that the sentence is no longer drawn from a set of seven constants; it is
drawn from a set of seven constants **and one bounded template**.

**Why the template does not reopen ADR-A009's ban.** ADR-A009 rejected composed flags because they
were *"non-deterministic in practice and unbounded in scope"*. The reason was never that a sentence
must be a literal; it was that a sentence must be reproducible and must not be able to say anything
the engine did not verify. The S1 template has neither defect:

- **Deterministic (P8).** Every slot — `{member}`, `{class}`, `{walked parents}`, `{trait|parent}`,
  `{declaring FQCN}`, `{path}`, `{line}`, `{applier}` — is filled from a fact `AncestryResolver`
  read from a file in the reviewed tree. Same tree, same bytes. The engine has interpolated exactly
  this tuple onto stderr deterministically since M4.
- **Bounded.** The slots are fixed by the template in `AssumptionWriter::STATEMENTS`; the values
  are names, paths and line numbers; no slot can carry prose, a slice, or anything the walk did not
  check. The "or in its parent" clause is present only when a parent was actually walked, so the
  sentence never asserts anything about a file that was not opened.

**The template is the bound.** The contract is not "payload is one of eight strings"; it is
"payload is one sentence produced by the catalogue's fixed statements, of which one is a template
whose slots are named in ADR-A009 and whose renderer is `inheritedMemberStatement()`". A second
template would need its own ADR-A009 entry and the same argument made again on its own facts; this
addendum clears one template, not the mechanism.

**What would have bumped.** A payload that carried a slice beside the sentence, a second sentence,
a structured object, or a slot filled from anything other than a verified file fact. None of these
is what was built, and the conformance test still validates the payload as a string.

**Cost, stated.** 62–63 tokens per S1 item, the longest flag in the catalogue by a factor of three.
Measured on the three recorded commits: D1 `ec92403` 606 → 730, `ee5a2e6` 1078 → 1393, `407c110`
unchanged. Whether that buys anything is the scoring question, not this one.

