# Architecture Freeze Review · 04 — ACP-01

Independent review of **ACP-01** ("NamedReference is one assertion kind produced by two different
moves") against the Architecture Repository at freeze review 03 and against the research repository.
The proposal was not trusted; each of its three claims was checked against the actual documents, and
one further finding was made independently of it.

**Verdict: ✅ Architecture Patch Approved (Freeze Review 04).**

---

## 1 · Is the inconsistency real?

Yes — all three symptoms, verified in the documents, not from the proposal's account of them.

| Claim | Verified against | Result |
|---|---|---|
| One kind, two extractors | `01-architecture.md` §3.3 module table said `OwnFileAssertionExtractor` emits "`NamedReference` (same-file) for sibling members"; the forms subsection immediately below listed `$this->method(` — same-class sibling, `upsertFromPlaid`, Exp 1 — under **`NamedReferenceAssertionExtractor`** | **Real.** Two extractors specified to emit the same assertion for the same case |
| One kind, two resolvers | §3.3: `OwnFileResolver` fetches "named sibling members"; `NamedReferenceResolver` resolves via `ClassLocator`. Dispatch is fixed as a `match` on `AssertionKind` (freeze review O3, ADR-A003) | **Real.** The `NamedReference` arm has two destinations and no way to choose |
| Priority band unresolvable | §3.4 band 2 read "Own-file slices … named siblings" (move vocabulary), band 4 "Named-reference slices" (kind vocabulary); `BundleItem` carries lever, reason, `assertion_kind`, provenance, payload, tokens — no producing move | **Real.** Exp 1's `upsertFromPlaid` (band 2) and `PlaidAccount` model surface (band 4) arrive at the enforcer indistinguishable |

ACP-01's root-cause diagnosis is also correct and better stated than the contract was: the priority
table was written in the vocabulary of **moves**, the bundle records the vocabulary of **kinds**, and
the two are 1:1 for three kinds and 1:2 for `NamedReference`. Confirmed defect of my own drafting —
introduced at freeze review 03 when the four recognised reference forms (correction B3) were added
under the `NamedReferenceAssertionExtractor` heading while the module table already assigned the
sibling case to `OwnFileAssertionExtractor`.

## 2 · Can the contract be implemented as written?

No. `Pipeline\DiscoverContext`'s `match` cannot be written without inventing a rule for which
resolver a `NamedReference` goes to, and `BudgetEnforcer` cannot band an item without inventing a
rule for which band a `named_reference` belongs in. Both inventions would be unevidenced behaviour in
the two places the architecture insists behaviour is stated, not inferred. Stopping was correct.

## 3 · Must the architecture change?

Yes — and the research decides the shape, so this is not a design choice:

- `docs/02-discovery/context-types.md` lists **type 1 same-file** and **type 2 named collaborator
  (depth 1)** as *distinct* context types.
- `docs/02-discovery/discovery-moves.md` lists **read-own-file** and **fetch-collaborator** as
  *distinct* moves, each with its own earning experiment.
- `05-traceability.md` §3 already mapped those two types to two different resolvers.

The contract collapsed two research-distinct types into one assertion kind. The patch restores a
distinction the research already draws; it does not introduce one.

**ACP-01's proposed fix is accepted as written.** Its rejected alternatives were re-checked and the
rejections hold: restating the priority table by kind fixes only symptom 1 and knowingly demotes Exp
1's own-file substrate into the band dropped first; giving `BudgetEnforcer` the changed-file set puts
diff knowledge into `Domain\Bundle`, which §3 forbids; recording the producing move adds an output
field R4 does not require.

One judgement ACP-01 correctly referred up: **`bundle_version` stays `1`.** The change is additive,
and no bundle has been emitted by a working tool — there is no published v1 to break.

## 4 · Second finding, made independently of ACP-01

ACP-01's M4 notes flagged, as an observation, that a flagged item's provenance carries no member. That
is a genuine contract inconsistency it under-reported: **ADR-A009 stated flags carry "provenance
(path, member)"**, while `Domain\Assertion` carries no enclosing-member name and the flag path has no
slicer. The claim was unsatisfiable, and the illustrative sample in `03-interfaces.md` §2 showed
`"member": "syncFromResponse"` on a flagged item.

The research asks only that the assumption be stated and attributed to its file
(`fetch-vs-flag.md`); it never requires member-level attribution on a flag. So the smallest correction
is to the *claim*, not to the domain: ADR-A009 now says path and line span always, member only when
the failing assertion already names one, and the sample was corrected. No field, collaborator, or
signature changed.

## 5 · The patch

| Document | Edit |
|---|---|
| `01-architecture.md` §3.1 | `AssertionKind` gains `SameFileReference`; noted that one kind per move means a kind names its resolver and its band |
| `01-architecture.md` §3.3 | `OwnFileAssertionExtractor` emits `SameFileSymbolAbsence` + `SameFileReference`; `NamedReferenceAssertionExtractor` is cross-file only; the `$this->method(` row moved out of the forms table, with the type-1/type-2 citation. **The forms table therefore holds three rows, not the four fixed at freeze review 01 (B3)** — the fourth became `SameFileReference`. Every live statement of that count was updated (`06-acceptance.md`, ADR-A003, the reviewable document); the counts in freeze reviews 01–02 are left as written, being accurate history |
| `01-architecture.md` §3.4 | Bands restated in `AssertionKind` + `Lever` — the two fields `BundleItem` already carries |
| `03-interfaces.md` §2 | `assertion_kind` gains `same_file_reference`, inserted into the fixed item order after `same_file_symbol_absence`; `bundle_version` stays 1; provenance rule for fetched vs flagged; flagged sample corrected |
| `03-interfaces.md` §4 | The five-arm dispatch map written out beside `LeverPolicy` |
| `ADR-A003` | Extractor/kind table corrected; a paragraph recording the re-partition and why the four-step gate is **not** triggered |
| `ADR-A009` | Flag provenance claim corrected (finding 4) |
| `05-traceability.md` | R1 and the move/type tables carry their kinds; P1 row notes the one-to-one partition |
| `06-acceptance.md` | `experiment-01` entries tagged with their kinds and bands; two unit tests added |
| `README.md` | Status → implementation ready at freeze review 04 |

Untouched: `00-research-verification.md`, `02-project-structure.md`, `04-diagrams.md`,
`evidence-gaps.md`, ADR-A001, A002, A004–A008.

## 6 · No new capability

- **No new module, port, class, CLI option, input, output channel, stage, premise, or `BundleItem`
  field.** Still **25 classes + 5 interfaces**, four extractors, three resolvers, five moves.
- **No new assumption.** AA1–AA10 unchanged. The patch's justification is `context-types.md` types 1
  and 2 plus Exp 1's `upsertFromPlaid` — evidence already cited in the contract.
- **Nothing gained a behaviour.** `OwnFileAssertionExtractor` emits the same assertions under a
  distinct label; `NamedReferenceAssertionExtractor` loses a row it was never solely responsible for.
  No new source is fetched, no new item is emitted, no premise fires differently.
- **Scope unchanged.** R1–R5 and X1–X5 as frozen. Depth one intact — `SameFileReference` resolves
  *within* the changed file, so it cannot even reach another file.

## 7 · Consistency review

1. **One kind per move.** Five kinds, five moves, one resolver destination each: absence and
   same-file reference → `OwnFileResolver`; named reference → `NamedReferenceResolver`; changed
   signature → `CallerResolver`; premise → flag. The `match` is now total with no fall-through.
2. **Priority computable.** Every band is a function of `assertion_kind` + `lever`, both already on
   `BundleItem`. `BudgetEnforcerTest` is now writable, so P7's "visible drops" are checkable.
3. **Extraction unambiguous.** Exactly one extractor emits each kind; the forms table is cross-file
   only, and its heading now matches its contents.
4. **Ordering total.** `same_file_reference` has a fixed position in the item order, so output stays
   byte-stable (P8).
5. **Acceptance agrees with implementation.** Experiment 1's three fetch-expected entries now name
   their kinds and bands; the flag-satisfied caller (freeze review 03) is unaffected.
6. **Principles intact.** P1 strengthened — the kinds now mirror the research's own taxonomy. P3/P4,
   P5, P6, P7, P8, P10 unchanged; ADR-A005's slices-not-files boundary untouched.
7. **Traceability intact.** `SameFileReference` → R1 → Exp 1 (`upsertFromPlaid`), recorded in
   `05-traceability.md` §1 and §2.

## 8 · Note on process

ACP-01 was correct on the defect, correct on the root cause, correct to refuse to invent the missing
rule, and correct to keep the proposal out of the implementation repository (freeze review L4). Its
one under-call — treating the flag-provenance contradiction as an observation rather than a contract
defect — is now patched. The implementation repository remains the single consumer of this contract
and was not modified by this review.

**Implementation may resume** at D10 (`Assembly/ItemPriority`, `Assembly/BudgetEnforcer`) and M6,
against this patch. The M4 work already completed needs one change only: `AssertionKind` gains the
fifth case and the item order includes it — nothing built in M0–M4 is invalidated.

---

## 9 · Residual defect closed: correction O4 was only partially applied

Found while re-reading the patched contract, unrelated to ACP-01. Freeze review 01's correction O4
deleted `--out`, and [REVIEW-freeze-02.md](REVIEW-freeze-02.md) row 9 certified it — but the option
survived in three places that then contradicted the CLI contract in `03-interfaces.md` §1:

| Location | Was | Now |
|---|---|---|
| `01-architecture.md` §3.6, `Cli\DiscoverCommand` row | "writes the bundle to stdout or `--out`" | "writes the bundle to stdout" |
| `01-architecture.md` §4, data-flow stage 8 | `─▶ stdout | --out` | `─▶ stdout` |
| `Phase 1 Architecture.dc.html`, §4.2 request flow, step 09 | "to stdout or `--out`" | "to stdout" |

This is the same defect class ACP-01 was raised for — two statements inside one contract disagreeing,
with the implementer left to choose — and the module responsibility table is what Claude Code
implements from, so it would have produced an `--out` branch that §1 forbids and no fixture covers.
Freeze review 02 row 9 has been corrected to record the partial application rather than a clean ✔, so
the audit trail stays honest. No design change: three string deletions.

## ✅ Architecture Patch Approved (Freeze Review 04)

Patched at: freeze review 04, against `AliAlqrinawi/context-discovery-research` @ `main`
(tree `785e3eb811fc`).
