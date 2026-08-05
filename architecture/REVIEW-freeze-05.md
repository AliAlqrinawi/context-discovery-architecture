# Architecture Freeze Review · 05 — the empty-result case

Independent review of the M7 observation against the research repository and the Architecture
Repository at freeze review 04. The observation was not trusted; the case was reconstructed from the
documents, and both proposed options were tested against the evidence before either was accepted.

**Verdict: ✅ Architecture Patch Approved (Freeze Review 05).** One distinction stated; no seventh
premise; no new module, capability, or scope.

---

## 1 · Is this a real gap?

Yes, and it is a genuine contradiction inside the contract rather than an under-specification:

| Document | Says |
|---|---|
| `03-interfaces.md` §4, `AssertionResolver` | "`@return list<SourceSlice>` empty list ⇒ caller **must flag** instead (P10)" |
| `01-architecture.md` §4, stage 5c | "resolution failure ─▶ back to 5b as a flag (P10)" |
| `ADR-A009` | Six premises, closed. Every one requires its literal trigger. **None** has a trigger matching a search that completed with zero results |

So the resolver contract mandates a flag, and the catalogue makes that flag impossible to emit
truthfully. `call-sites-truncated` requires hitting `--max-call-sites`; `unresolved-reference` requires
a `NamedReference` that failed to resolve. Neither is true here. The implementation was right to stop
rather than emit a false premise.

The root cause is a single over-broad word. "Empty result" was written as one case when it is two:
**lookup failure** (the source should exist and could not be read) and **successful negative** (the
question was answered and the answer is "none"). The contract collapsed them, and P10's own wording —
"*fail* toward flagging" — only ever covered the first.

## 2 · Does the architecture already answer it?

No. Three places were checked and each is either silent or wrong on this case: `01-architecture.md`
§4 stage 5c (names only failure, then routes "empty" to it), `03-interfaces.md` §4 (mandates the flag),
and ADR-A009 (no matching trigger). `05-traceability.md`'s P10 row repeated the over-broad reading.

## 3 · Which option does the research support?

**Option 2, with a correction to how it is framed.** Option 1 is rejected on three independent grounds:

1. **It would state an untruth.** A premise is a claim the run could not verify. A caller search that
   completed with zero results verified the thing. There is no assumption to disclose.
2. **The spec counts that as a precision failure.** Its criterion is explicit that pulling material
   "just in case" is "a precision failure, not caution". An item asserting an assumption nobody is
   making is the same error in flag form — and it would appear on *every* signature change with no
   callers, inflating the flag count the scored run is meant to measure.
3. **No experiment earns it.** ADR-A009's gate requires an experiment per premise. None exists.

The positive case for Option 2 is in the research, not in convenience: `discovery-moves.md`'s fifth
move is **return-empty/flag**, and Experiment 2 earned "return empty" as a *correct outcome* — the
whole point of that experiment is that producing nothing is sometimes right. A resolver returning
nothing because there is nothing is that move, not a failure of it.

**One thing the observation did not consider, and the reason this is not simply "no change".** Zero
callers under a *bounded* scope could be read as leaving something unverified — callers may exist
outside `--caller-scope`. That would make a seventh premise defensible by analogy with
`call-sites-truncated`. It is rejected: Exp 4's minimum context is "a caller search for `reactivate(`
across `app/`", i.e. the research treats a search across `app/` as *the answer*, not as a partial one.
Treating it as partial would import a doubt no experiment recorded. The question is recorded instead as
a named under-build in `evidence-gaps.md` §4, with its trigger: an experiment showing a finding whose
call sites lie outside the searched scope.

## 4 · The patch

| Document | Edit |
|---|---|
| `03-interfaces.md` §4 | `AssertionResolver::resolve()`'s contract splits the empty return into (a) lookup failure ⇒ flag (P10) and (b) successful negative ⇒ no item, one stderr diagnostic. The resolver distinguishes them; the pipeline never guesses |
| `03-interfaces.md` §1, §2 | One CLI-contract line; the diagnostics paragraph names the one diagnostic class with no corresponding item, with its wording |
| `01-architecture.md` §3.3 | `CallerResolver` row states the zero-result outcome |
| `01-architecture.md` §4 | Stage 5c narrowed to failure; new stage 5d for the successful negative; a data-flow property recording why an empty result is not automatically a flag |
| `ADR-A009` | "A premise exists only where an unverified premise exists", and the explicit rejection of the seventh premise with both reasons |
| `05-traceability.md` | P10 row: failure routes to a flag; a successful negative does not |
| `06-acceptance.md` | `CallerResolverTest` gains the zero-result and unreadable-scope cases |
| `evidence-gaps.md` | **AA11** (the diagnostic line); the scope-limited premise recorded as an under-build with a trigger |
| `README.md` | Status → implementation ready at freeze review 05 |

Untouched: `00-research-verification.md`, `02-project-structure.md`, `04-diagrams.md`, ADR-A001–A008.

## 5 · No new capability

- **No new module, port, class, premise, CLI option, bundle field, or stage.** Still 25 classes,
  5 interfaces, 6 premises, 5 assertion kinds, 4 extractors, 3 resolvers.
- **No new output.** The stderr diagnostic channel already exists (freeze review L2). One message class
  was named on it; it is recorded as **AA11** because the research does not require it.
- **Scope unchanged.** R1–R5 and X1–X5 as frozen. The change *removes* an output the contract
  previously demanded; nothing is added.
- **No search behaviour changed.** `CallerResolver` still runs only for `ChangedSignature`, still
  bounded, still scoped.

## 6 · Consistency review

1. **P10 intact and now accurately stated.** "Fail toward flagging" applies to failures. Silence is
   still forbidden where something is unverified — the two P10 premises are untouched.
2. **The general rule covers all three resolvers**, so no per-resolver special case exists: a changed
   file with no `use` block is the same successful negative, and `NamedReference` resolving to nothing
   remains a failure (the reference is in the diff, so the source should exist) ⇒ still a flag.
3. **ADR-A009's gate is strengthened, not weakened.** The catalogue now states the *condition* for
   membership, so the next proposed premise is testable against a rule instead of a precedent.
4. **Precision preserved.** Experiment 2's almost-empty bundle is unaffected; Experiment 4's caller
   item is unaffected (callers exist there). The change only affects a case no fixture exercises today —
   which is why it must be specified rather than discovered.
5. **Verifiability preserved.** The diagnostic is what lets the harness distinguish "searched, found
   none" from "never searched" — the assertion Experiment 1's fixture already makes.
6. **Determinism intact.** No ordering, no budget, no schema change.

## 7 · How implementation should proceed

Resume at M7. For a `ChangedSignature` whose search completes with zero call sites: emit no bundle
item, emit no flag, write one stderr line, continue. Reserve the flag path for a search that could not
run or that hit `--max-call-sites`. Nothing built before M7 is invalidated; the M7 work already done
needs the failure/negative branch split, and `CallerResolverTest` gains two cases.

---

## ✅ Architecture Patch Approved (Freeze Review 05)

Patched at: freeze review 05, against `AliAlqrinawi/context-discovery-research` @ `main`
(tree `785e3eb811fc`).
