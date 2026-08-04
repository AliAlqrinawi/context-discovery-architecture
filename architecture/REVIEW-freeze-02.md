# Architecture Freeze Review · 02 — final

Second and final review, run against the **updated** Architecture Repository after all fourteen
corrections from [REVIEW-freeze-01.md](REVIEW-freeze-01.md) were applied. Scope of this review:
verify each correction landed, verify nothing else moved, and re-run the nine review questions
against the current documents.

---

## 1 · Corrections verified

| # | Correction | Landed in | Verified |
|---|---|---|---|
| 1 | `ItemPriority` drop order stated (4 bands, evidence per band, largest-first within a band) | `01-architecture.md` §3.4 | ✔ referenced from `05-traceability` P7 and `06-acceptance` |
| 2 | Acceptance diffs declared operator-supplied; harness fails loudly | `06-acceptance.md` §1, `02-project-structure.md` tree | ✔ |
| 3 | Four recognised reference forms, each cited to Exp 1 or 4; any other form yields nothing | `01-architecture.md` §3.3 | ✔ new `NamedReferenceAssertionExtractorTest` in `06-acceptance` |
| 4 | `filesUnder()` lexicographic; call sites in that order | `03-interfaces.md` §3, `ADR-A006`, `05-traceability` P8 | ✔ |
| 5 | `ResolvedAssertion` added; `assemble()` retyped | `01-architecture.md` §3.1, `02-project-structure.md`, `03-interfaces.md` §4 | ✔ tuple gone |
| 6 | `Ports\DiffParser` deleted; parser is `Discovery\Parsing\UnifiedDiffParser` | `01-architecture.md` §3.2–3.3, §3.5, `02-…`, `03-…`, `05-…` R1 | ✔ `Adapters\Diff/` removed everywhere |
| 7 | `Ports\TokenEstimator` deleted; `Assembly\TokenEstimate` | same set, `ADR-A007`, `05-…` R4 | ✔ `Adapters\Estimation/` removed everywhere |
| 8 | `supports()` dropped; explicit `match` on `AssertionKind` | `03-interfaces.md` §4, `ADR-A003` | ✔ |
| 9 | `--out` deleted | `03-interfaces.md` §1, `01-architecture.md` §5 | ✔ |
| 10 | A007 formula removed; one named ratio constant; ratio recorded as AA4 | `ADR-A007`, `05-traceability` §6 | ✔ no expression remains |
| 11 | `diagnostics[]` removed from the bundle schema | `03-interfaces.md` §2, review document sample | ✔ stderr-only, each with a flag item |
| 12 | *Repository* naming clarified | `02-project-structure.md` §5, `03-interfaces.md` §5 | ✔ |
| 13 | Provenance line required on the shipped copy of the contract | `02-project-structure.md` tree | ✔ |
| 14 | AA1–AA8 recorded | `evidence-gaps.md` §5, `05-traceability.md` §7 | ✔ |

Incidental consistency fixes made while applying the above, no design content: "Six groups" → "Seven
groups" (`02-project-structure.md` §2), module count 24+7 → **25 classes + 5 interfaces**, and the
reviewable document (`Phase 1 Architecture.dc.html`) brought in line with all of it.

## 2 · No new concepts introduced

Checked explicitly, because that was the binding constraint:

- **One new type**, `ResolvedAssertion` — a value object for a tuple the contract already described.
  It adds no capability and appears in no output.
- **Nothing else added.** Two interfaces removed, one CLI option removed, one schema field removed,
  one class relocated (`Adapters\Diff` → `Discovery\Parsing`), one class renamed and relocated
  (`CharRatioTokenEstimator` → `Assembly\TokenEstimate`).
- **No new assumption.** AA1–AA8 are the *existing* choices reclassified as assumptions; none was
  created by this revision. AA4 was strengthened by *removing* the invented formula from the ADR.
- **No boundary change.** R1–R5 unchanged. X1–X5 unchanged. No premise added to A009. No new input,
  no new output channel, no new stage — the pipeline is still the spec's nine responsibilities.

Net module count: **25 classes + 5 interfaces** (was 24 + 7).

## 3 · The nine review questions, re-run

1. **Every module supported by Phase 0 evidence.** ✔ `05-traceability.md` §1–§3 covers all 25
   classes and 5 interfaces; the two removals shrank the surface, so no new module needs evidence.
   `Discovery\Parsing\UnifiedDiffParser` traces to R1 (Exp 1); `Assembly\TokenEstimate` to R4/P7.
2. **No unnecessary abstraction.** ✔ Every surviving interface has more than one reason to exist:
   four ports invert I/O; `BundleWriter` has two runtime-selected implementations. The two
   single-implementation pure-function ports are gone.
3. **No hidden over-engineering.** ✔ The `supports()` probe — the one plugin-shaped seam — is gone;
   dispatch is one `match`. No cache, index, graph, queue, worklist, dispatcher, or parallelism.
4. **No implementation detail leaked.** ✔ The `ceil(strlen/4)` expression was the only code-level
   leak and is removed. `token_get_all` and the CLI defaults remain, correctly: the first pins a
   dependency posture (ADR-A004), the second is the public contract.
5. **Every interface has a single responsibility.** ✔ Five ports, one job each. `MemberSlicer`'s four
   methods are one responsibility (PHP text → member spans) and all four are consumed.
6. **Dependency direction correct.** ✔ `Domain` imports nothing. `Discovery`/`Assembly`/`Pipeline`
   name no adapter. `Adapters` see Ports and Domain only. `Cli\Wiring` is the single construction
   point. Relocating the parser into `Discovery\Parsing` keeps the arrow inward; no cycle exists.
7. **Every ADR justified by research evidence.** ✔ A001–A009 each cite a spec clause, a principle, or
   a named experiment. A002, A003, A006 and A007 carry the freeze amendments. Assumption-grade
   content is now labelled AA1–AA8 rather than argued as evidence.
8. **Nothing violates Phase 1 scope.** ✔ Scope moved only downward: one CLI option and one schema
   field fewer. X1–X5 remain structurally unreachable — no forward-following module, no worklist that
   could hold pending references, no config/migration resolver, no severity type, no HTTP client.
9. **Nothing missing that would block implementation.** ✔ The four blocking omissions are closed: the
   drop order is stated, the fixture prerequisite is declared with a loud failure mode, the
   reference-recognition contract is enumerated with citations, and the ordering contract makes
   determinism achievable. `assemble()` is fully typed.

## 4 · Standing conditions carried into implementation

Not defects — the honest boundary of what is being handed over:

- The hypothesis remains **open**. Phase 1 makes the scored A-vs-C run possible; it does not perform
  it, and no module encodes an expectation about the outcome.
- **AA1–AA8** are assumptions, not evidence (`evidence-gaps.md` §5). Each names where it is used.
- **Four under-builds with named triggers**: config/schema by flag (X3), grep instead of a call graph
  (R3), the one-constant token estimate (A007), unified-diff format only.
- **The research repository still lacks `docs/03-phase0/summary.md` and `CONTRIBUTING.md`**, and its
  README's directory map does not match its tree (`evidence-gaps.md` §1). That is research-repository
  work; it does not block implementation.
- **First run after build is the sloppy Experiment 5 commit** (`06-acceptance.md` §4). If it raises an
  assertion kind the four extractors cannot express, the bundle shows the gap and the question goes
  back to the research repository — it is not patched here.

---

## ✅ Architecture Frozen

All fourteen approved corrections are applied and verified. No new concept, capability, assumption,
module, or scope was introduced; the surface is strictly smaller than at review 01. Traceability is
intact — every module maps to a requirement and an experiment, and every unproven choice is labelled
as an assumption. Implementation may begin against this contract, in the order given in
[README.md](README.md).

Frozen at: freeze review 02, against `AliAlqrinawi/context-discovery-research` @ `main`
(tree `785e3eb811fc`).
