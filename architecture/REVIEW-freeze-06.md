# Architecture Freeze Review · 06 — the caller-search failure case

Independent review of the M7 observation against the research repository, the Architecture Repository
at freeze review 05, and freeze review 05 itself. The implementation's current behaviour was not
trusted; both readings of the trigger were tested against the documents.

**Verdict: ✅ Architecture Patch Approved (Freeze Review 06).** One premise added; no module, no
capability, no scope change.

---

## 1 · The question, answered

**The trigger means only `NamedReference` lookup failure.** Not "any lookup failure". Three
independent reasons, all in the current documents:

1. **The trigger text names the kind.** ADR-A009: "A `NamedReference` assertion resolved to nothing".
2. **The fixed statement names it too** — "ASSUMPTION: **named reference** could not be resolved on
   disk; contract unverified". That sentence is *false* of an unreadable caller scope: nothing about a
   named reference failed. The catalogue's whole purpose is that a flag says something exactly true.
3. **The provenance rule depends on it.** ADR-A009 (freeze review 04) cites `unresolved-reference`
   "from a `NamedReference`" as the case where a flag carries a member. A caller-search failure has no
   such member, so the two cases are already treated as different upstream.

So the implementation's current output is a defect: factually true in substance, factually wrong in
wording. It should not ship.

## 2 · Is this a real gap?

Yes. Freeze review 05 established the *rule* — a lookup failure must flag — and its §6 point 2 said
the rule "covers all three resolvers". It did not check that the catalogue had a premise for each
resolver's failure. It does not: `CallerResolver` owns `call-sites-truncated` (bound hit) but has no
entry for "search could not run".

The architecture therefore answers this case **partially and contradictorily**: P10 and freeze review
05 demand a flag; the catalogue offers none that is true. That is the same defect class as ACP-01 and
the freeze-05 question — a rule stated at one level, unsupported at another. My omission, introduced at
freeze review 05 when the failure/negative split was written without auditing the catalogue against it.

## 3 · The patch

A seventh premise, symmetric with the sibling `CallerResolver` already had:

| Premise | Trigger | Statement | Basis |
|---|---|---|---|
| `caller-search-failed` | A caller search **could not run** — scope prefix unreadable or absent | "ASSUMPTION: callers of this signature could not be searched; scope unreadable" | P10, ADR-A006 |

Standing: P10-derived, exactly like `unresolved-reference` and `call-sites-truncated`, so it falls
under existing assumption **AA5** rather than creating a new one. No new AA was needed.

The general rule is now stated in ADR-A009 so the next such question answers itself: **one P10 premise
per lookup that can fail, never a shared one.** Two lookups can fail — `NamedReferenceResolver`'s
(PSR-4 locate + member slice) and `CallerResolver`'s (scope scan) — and each has its own true
statement.

Note this is *not* the seventh premise freeze review 05 rejected. That one was for a search returning
**zero results**, where nothing is unverified. This one is for a search that **could not run**, where
the callers are genuinely unchecked. The two are now adjacent in the catalogue and explicitly
distinguished, because the difference is the whole point.

### Documents updated

| Document | Edit |
|---|---|
| `ADR-A009` | `caller-search-failed` added to the trigger table and the statement table; `unresolved-reference` marked resolver-specific; the "one P10 premise per lookup" rule; two new rejected alternatives (reuse; broadening to "any lookup failure") |
| `01-architecture.md` §3.3 | `CallerResolver` row: cannot-run ⇒ `caller-search-failed`; extractor row notes three P10 failure premises |
| `03-interfaces.md` §4 | `resolve()`'s failure case names the per-resolver premise and forbids sharing |
| `05-traceability.md` | P10 row: each failure uses its own resolver's premise; R5 row now says seven premises |
| `06-acceptance.md` | `CallerResolverTest`: unreadable scope ⇒ `caller-search-failed`, never `unresolved-reference` |
| `evidence-gaps.md` | AA5 extended to name the third P10 premise |
| `README.md` | Status → implementation ready at freeze review 06 |

Untouched: `00-research-verification.md`, `02-project-structure.md`, `04-diagrams.md`, ADR-A001–A008.

## 4 · No new capability

- **No new module, port, class, resolver, CLI option, bundle field, stage, or input.** Still 25
  classes, 5 interfaces, 5 assertion kinds, 4 extractors, 3 resolvers. Premises: 6 → **7**.
- **No new assumption.** AA1–AA11 unchanged; the new premise sits under AA5.
- **No new behaviour.** The flag was already mandated by P10 and freeze review 05; the patch only makes
  it *sayable*. Nothing new is searched, fetched, or emitted — one existing flag path gains an accurate
  sentence.
- **Scope unchanged.** R1–R5, X1–X5 as frozen. ADR-A006's grep stays bounded and scoped.

## 5 · Consistency review

1. **Catalogue closed and complete.** Every lookup that can fail has exactly one premise; every
   premise has exactly one trigger and one true statement. The audit that was missing at freeze review
   05 is now recorded as a rule.
2. **Freeze review 05 intact.** Zero results still yields no item and one diagnostic; the failure and
   negative cases are adjacent in the catalogue and explicitly contrasted, so neither can absorb the
   other.
3. **ADR-A009's gate honoured.** The premise is P10-derived, the same basis as its two siblings, and is
   labelled as such rather than presented as experiment-earned.
4. **Provenance rule holds.** `caller-search-failed` carries path and line span, no member — consistent
   with freeze review 04, since the assertion names no member.
5. **Precision unaffected.** No Phase 0 fixture has an unreadable scope, so no expectation changes;
   Experiment 4's caller item is untouched.
6. **P6/X4 intact.** A fixed sentence, no severity, no judgement.

## 6 · How implementation should proceed

Resume at M7. Branch `CallerResolver`'s outcome three ways: slices found ⇒ fetched item; search ran and
found nothing ⇒ no item, one stderr diagnostic (freeze review 05); search could not run ⇒
`caller-search-failed` flag (this review). Replace the `unresolved-reference` emission currently in the
scope-failure path. `unresolved-reference` remains exclusive to `NamedReferenceResolver`. Nothing built
before M7 is invalidated.

---

## ✅ Architecture Patch Approved (Freeze Review 06)

Patched at: freeze review 06, against `AliAlqrinawi/context-discovery-research` @ `main`
(tree `785e3eb811fc`).
