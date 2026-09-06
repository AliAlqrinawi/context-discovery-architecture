# ADR-A015 · ADR-A010's revisit trigger is evaluated and declined — the depth boundary stands

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · formally evaluates the trigger [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) set
- **Implements:** P3, P10, X2, ADR-A001, ADR-A003, ADR-A009, ADR-A010
- **Changes:** **nothing.** No production code, no boundary, no premise, no lever, no schema. This ADR
  records a decision *not* to change, and the measurements that make that the right answer.

## Why this decision exists

ADR-A010 fixed resolution at one file beyond the changed file and named the condition for reopening:

> **D2 is relaxed when, and only when, an ADR-001-grade experiment exists whose hand-written answer
> key names a member the changed file's directly-referenced class does not declare** — that is, a
> finding a reviewer needed, which only an ancestry read could have supplied.

M7 ran Experiment 5 on a real pull request and produced row **E5.4** — `App\Traits\ApiResponse::success`
— which appears to be exactly that. M8 exists to test whether it is.

## The trigger is met in letter

Verified against `abouelsid-backend` @ `ec92403`:

| Question | Answer |
|---|---|
| What class does the changed file directly reference? | `App\Http\Controllers\Controller` — imported at `MenuPdfController.php:5`, and `extends Controller` at line 11 |
| Where is it? | `app/Http/Controllers/Controller.php` — placeable, one hop |
| Does it declare `success`? | **No.** The file is six lines: `abstract class Controller { use ApiResponse; }`, and declares **zero** functions |
| Where is `success` declared? | `app/Traits/ApiResponse.php:9` |
| By what mechanism? | A **trait applied to the parent**. Guaranteed by PHP semantics, not inferred |
| How many files from the changed one? | **three**: `MenuPdfController` → `Controller` → `ApiResponse` |

A hand-written key from an ADR-001-grade experiment names a member the directly-referenced class does
not declare. **The letter of the trigger is satisfied.**

## It is not met in substance, and the measurements say why

### 1 · The relaxation the trigger licenses does not reach E5.4

The only bounded relaxation available is one additional hop. `success` is **three** files away, behind
an abstract parent *and* a trait. Fixture `experiment-08` row **X8.6** reproduces the shape exactly —
`ViaParent extends Controllerish { use Greeter; }` — and it flags today and would still flag after a
one-hop relaxation.

### 2 · E5.4's actual blocker is extraction, not depth

`$this->success(...)` is a `$this->method(` form. `OwnFileAssertionExtractor` skips it because
`success` is not a member of the changed file, so **no assertion is ever formed**. Verified in M7's
captured output: zero bundle items and zero diagnostics mention `success` or `ApiResponse`.

Relaxing a resolution bound cannot help a reference that resolution never sees. Fixture row **X8.10**
isolates this: it is a **silent omission**, which P10 rates worse than a flag, and it is an
`AssertionExtractor` question — ADR-A003 gated, and adjacent to M7's G1.

### 3 · One hop would fix none of the measured false positives

M7's nine false-positive flags, traced to their declarations:

| Subject | Immediate parent | Declared on the parent? | Actually declared at |
|---|---|---|---|
| `Setting::updateOrCreate`, `::where`, `::create` | `Model` | **no** | `Eloquent/Builder.php` |
| `MediaItem::where`, `::create` | `Model` | **no** | `Eloquent/Builder.php` |
| `User::create` | `Authenticatable` | **no** | `Eloquent/Builder.php` |

**0 of 9.** Every one is Eloquent dynamic dispatch at three or more hops. `Package::query` — the case
ADR-A010 itself used as its minimal decision case — is a one-hop case, and it does not occur in the
real diff at all.

### 4 · Across an entire real application, one hop into project code fixes nothing

Every static call on a project class in `abouelsid-backend/app` (190 sites):

| Category | Count | Reached by one hop? |
|---|---|---|
| member declared in the named class | 73 | already fetched today |
| **one hop to a project parent or the class's own trait** | **0** | yes — but there are none |
| one hop to a **dependency** parent (`XResource::collection`) | 21 | yes — see below |
| Eloquent dynamic dispatch (`create`, `findOrFail`, `orderBy`…) | 96 | no |

The case `experiment-08` rows X8.3 and X8.5 were written to represent — a member inherited once from
project code — has **zero occurrences** in a real Laravel application of this size.

## Decision

**ADR-A010's boundary stands unchanged.** Resolution opens at most one file beyond the changed file;
`extends`, `use <Trait>` and `@mixin` are not followed.

The trigger is recorded as **evaluated and declined**, not as unmet: E5.4 satisfies its wording, and
the reason for declining is that the remedy it licenses demonstrably fixes nothing the evidence
measures. A future proposal must clear the bar again on its own evidence, not cite E5.4 as
already-spent.

## Options considered

| | Option | Verdict |
|---|---|---|
| **A** | Keep ADR-A010 unchanged | **Chosen** |
| **B** | Permit exactly one additional hop | Rejected — fixes 0 of 9 measured false positives, 0 real project-parent sites, and not E5.4. Would change behaviour with no evidence behind it |
| **C** | Let framework knowledge answer the member without opening more files | Rejected — **fails precision**. For a facade the positive evidence is a `@method static` tag; `Model` carries no `@method` and no `@mixin`, so nothing inside one file distinguishes `Package::create` from `Package::activatte`. Settling on "extends Model" alone would swallow the typo, which is M5's risk R1 and M1's guard S08.2 |
| **D** | A new extraction/resolution move | Rejected for M8 — this is what E5.4 and X8.10 actually need, and it is ADR-A003-gated and adjacent to M7's G1, which M8 was instructed to keep separate |
| **E** | Resolve **OQ6** instead — make the flag's statement true at depth one | **Not taken now, and strengthened as the recommended next step.** ADR-A010 already names it as the alternative it does not close. It costs no depth and no new premise, and it would make all 117 of these flags *true* rather than false. It changes an ADR-A009 fixed statement, so it needs its own decision |

### On Option B and the 21 `XResource::collection` sites

The one real one-hop population is `UserResource::collection` and its 20 siblings:
`UserResource extends JsonResource`, and `collection()` **is** declared one hop away at
`vendor/laravel/framework/src/Illuminate/Http/Resources/Json/JsonResource.php:86`. Under ADR-A012 and
ADR-A013 the parent is a dependency, so one hop would convert 21 false-positive flags into 21 cited
settlements at **zero token cost** — a genuine precision gain.

It is not taken, for a reason that matters more than the gain: **this population was discovered while
measuring, after the answer key was written.** `experiment-08`'s key asks for the *project*-parent
case (X8.3, X8.5), which has zero real occurrences; it does not contain a dependency-parent row at
all. Writing a rule to fit something found after the key is the confirmation bias ADR-001's key-first
rule exists to prevent. The finding is recorded as **new, quantified evidence** and needs its own
key-first experiment.

## Consequences

**Positive**

- The boundary is now backed by measurement rather than by the absence of a counter-example, and the
  trigger has been exercised once without being spent loosely.
- Three distinct blockers that were being discussed as one are now separated and individually
  measured: **depth** (X8.3/X8.5, 0 real occurrences), **dynamic dispatch** (96 real sites, no file
  declares the member at any depth), and **extraction** (E5.4/X8.10, no assertion formed).
- `experiment-08` gives every future depth proposal a ready-made, framework-free test bed with both
  sides of the boundary keyed.

**Negative, stated plainly**

- M7's G2 (9 false flags on that PR, 96 sites application-wide) and G5 (a silent omission) are
  **unfixed**, and this ADR does not put them on a path — it establishes that the path they were
  assumed to be on is the wrong one.
- The 21 dependency-parent sites keep flagging falsely although a bounded fix exists. That is a cost
  paid deliberately to keep the key-first rule intact.

## Recorded evidence gaps

| Gap | What it needs |
|---|---|
| **One hop to a dependency parent** — 21 real sites, precision-only, zero token cost | A key-first experiment that contains the case **before** the key is written |
| **G2 · Eloquent dynamic dispatch** — 96 real sites | No file declares the member at any depth (`experiment-08` X8.7 proves the mechanism). Needs framework knowledge that can distinguish a real Eloquent member from a typo, which nothing in one file can do |
| **G5 · `$this->inheritedMember()`** — silent omission, P10 | An extraction change, ADR-A003 gated |
| **OQ6** — the fixed statement is false for every case above | Its own decision; ADR-A010 already names it |

## Related

- [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md) — the boundary and the trigger.
- [ADR-A009](ADR-A009-premise-catalogue.md) — the fixed statement OQ6 would correct.
- [ADR-A003](ADR-A003-closed-move-set.md), [ADR-A001] — the four-step gate and the key-first rule.
- `docs/research/M7-experiment-05.md` and `docs/research/M8-depth-boundary-revisit.md`;
  fixtures `experiment-05/`, `experiment-08/`.
