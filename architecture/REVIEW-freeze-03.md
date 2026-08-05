# Architecture Freeze Review · 03 — implementation blockers closed

Third and final review. Scope: the two blockers raised by the implementation-readiness review
(C1, C2), the smallest corrections for each, and a consistency pass over the whole contract.
No module, capability, responsibility, or scope changed; the pipeline is untouched.

---

## C1 · The recognition contract for `UnverifiablePremise` was incomplete

### Root cause
[ADR-A009](decisions/ADR-A009-premise-catalogue.md) specified the **output** side of the flag path —
six premises, one fixed statement each — and `LeverPolicy` specified the **lever**. Neither specified
**detection**: what in the diff causes a premise to be emitted. `01-architecture.md` §3.3 described
the extractor as emitting "from a closed catalogue" without saying *when*. So the catalogue was closed
on its vocabulary and open on its trigger — the one gap that looks closed on a read-through.

### Why implementation could not proceed
Emitting a premise without a stated trigger requires the implementer to invent a rule for "this code
assumes a transaction" — i.e. to introduce judgement into a tool whose architecture forbids it
(P6, X4) and to introduce a behaviour traceable to no experiment, breaking the evidence-before-
architecture rule. Refusing to invent it was correct.

### Correction applied
ADR-A009 gains a **trigger column**: a premise is emitted **if and only if** its literal trigger is
present. Each trigger reads only the changed region and the changed file's own text — R1's existing
load, so no new input, module, or pass:

| Premise | Trigger | Earned by |
|---|---|---|
| `surrounding-transaction` | ≥ 2 persistence-write calls in the region *and* no transaction opened in the enclosing member | Exp 1 |
| `atomic-lock-store` | region names `Cache::lock(` | Exp 3 |
| `schema-index-support` | region names `lockForUpdate(` or `sharedLock(` | Exp 3 |
| `data-state-after-behaviour-change` | a **removed** `use <Trait>;` line inside a class body | Exp 3 |
| `unresolved-reference` | a `NamedReference` resolved to nothing | P10 |
| `call-sites-truncated` | caller search hit `--max-call-sites` | P10, A006 |

Two widenings are marked as assumptions rather than evidence: the persistence-write token list
(**AA9**) and trait removal generalised beyond `SoftDeletes` (**AA10**).

Two precision guards are stated with the triggers: Experiment 2 fires **no** trigger (its trace
logging names no lock, removes no trait, adds no write), so its bundle stays almost empty; Experiment
4 asks for no premise, so if the transaction trigger fires there the *trigger* is wrong and the
question returns to the research repository rather than being tuned in the code (ADR-A003).

### No new capability
The extractor's inputs, outputs, module, and lever are unchanged. Six premises before, six after. What
changed is that a rule the implementer would otherwise have invented is now written down and cited.

---

## C2 · Experiment 1's acceptance criterion demanded output the contract forbids

### Root cause
`06-acceptance.md` transcribed each experiment's step-4 minimum-context list as though every entry
were something the tool must **fetch**. Experiment 1's list includes "the single caller of
`syncFromResponse` … resolves the transaction question definitively" — but that is what a *human*
assembled by hand. R3 restricts the caller grep to **changed signatures**, and `syncFromResponse`'s
signature did not change. So the acceptance contract required a caller fetch that the implementation
contract forbids: a direct contradiction, and the fixture could never pass.

### Why implementation could not proceed
Either document could be satisfied, never both. Choosing between them is an architectural decision, not
an implementation detail — and guessing would have meant either extending R3's grep beyond changed
signatures (scope expansion, and straight into the reverse-graph machinery `fetch-vs-flag.md` exists
to avoid) or silently weakening the acceptance test.

### Correction applied
The research settles it, so no design choice was needed: `fetch-vs-flag.md` names the transaction
question the canonical **flag** case, and the implementation spec's success criteria count flags "as a
catch for the flag-type findings (e.g. the transaction assumption)".

Every expectation entry is now marked **`fetch-expected`** or **`flag-satisfied`**, and the harness
compares against the mark. For `experiment-01`: the `use` block + `syncFromResponse` body, the model
surface, and `upsertFromPlaid` are `fetch-expected`; the caller is `flag-satisfied` by the
`surrounding-transaction` premise, and the fixture additionally asserts that **no caller grep ran**.
`experiment-03`'s premises are marked `flag-satisfied` (X3); `experiment-04`'s caller grep stays
`fetch-expected`, because there the signature genuinely changed.

### No new capability
No resolver gained a trigger, no grep gained a scope, nothing new is fetched or emitted. One
documentation-level marking was added so the two contracts state the same thing.

---

## Documents updated

| Document | Change |
|---|---|
| `decisions/ADR-A009-premise-catalogue.md` | Trigger column (the recognition contract), AA9/AA10 notes, the two precision guards |
| `01-architecture.md` | §3.3: premise-trigger pointer with the "no new input" property; "the caller is fetched only for a changed signature"; extractor row now cites the trigger rule |
| `06-acceptance.md` | `fetch-expected` / `flag-satisfied` marking rule; experiment-01, -03, -04 rows restated; two unit tests added (trigger table coverage; caller-grep restriction) |
| `05-traceability.md` | R3 row: signature-changes-only, transaction is flag-type; R5 row: triggers; move table row restated |
| `evidence-gaps.md` | AA9, AA10 added to §5 |
| `README.md` | Status → implementation ready; review index |
| `Phase 1 Architecture.dc.html` | Premise-trigger note in the pipeline diagram; acceptance card marking note |

Unchanged: `02-project-structure.md`, `03-interfaces.md`, `04-diagrams.md`, ADR-A001–A008. No file
gained a module, an option, a field, or a stage.

---

## Consistency review

1. **Scope.** R1–R5 and X1–X5 untouched. No input, output channel, port, class, or stage added.
   Module count remains **25 classes + 5 interfaces**.
2. **Traceability.** Every trigger names its experiment; the two widenings are labelled AA9/AA10 in
   `evidence-gaps.md` §5. The traceability tables now agree with ADR-A009 and with `06-acceptance.md`.
3. **No judgement introduced.** Triggers are literal token or diff-line tests. No severity, no
   ranking, no confidence, no inference — P6 and X4 hold.
4. **Depth one intact.** Triggers read the changed region and the changed file's own text only; no
   resolved file is re-read (P3, P4).
5. **Fail-to-flag intact.** `unresolved-reference` and `call-sites-truncated` remain the P10 path, now
   with their triggers stated rather than implied.
6. **Precision intact.** Experiment 2's almost-empty bundle is preserved *by the trigger set*, not by
   hope; Experiment 4 is the declared guard for the one trigger that is wider than its finding.
7. **The two contracts now agree.** For every fixture entry, what the acceptance test expects is
   something the implementation contract can produce — and the mark says which lever produces it.
8. **Falsification discipline preserved.** If a trigger misfires on a Phase 0 fixture, the recorded
   response is to return the question to the research repository, not to tune a threshold.

---

## ✅ Implementation Ready

Both blockers are closed with the smallest corrections available: one trigger column and one
expectation marking. Nothing was redesigned, no capability or module was added, no responsibility
widened, and the pipeline is unchanged. Two new architectural assumptions (AA9, AA10) are recorded as
assumptions rather than presented as evidence; everything else traces to a named experiment or
principle in the research repository.

Ready at: freeze review 03, against `AliAlqrinawi/context-discovery-research` @ `main`
(tree `785e3eb811fc`).
