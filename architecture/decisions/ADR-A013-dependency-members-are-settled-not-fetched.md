# ADR-A013 · A declared dependency member is settled, not fetched — and an undeclared one still flags

- **Status:** Accepted
- **Date:** 2026-09-05
- **Phase:** Phase 1 · completes the ownership rule [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) began
- **Implements:** R2, R4, P6, P7, P8, P9, P10, ADR-A003, ADR-A005, ADR-A009, ADR-A011, ADR-A012
- **Adds:** nothing. No assertion kind, no premise, no lever, no bundle field, no `bundle_version`
  change, no port, no module. One guard widened, one settlement sentence added.

## Why this decision exists

ADR-A012 stopped a **bare** dependency class from being dumped into the bundle. It deliberately left
the **member** case alone, and recorded the inconsistency:

> A bare dependency class is withheld; a dependency *member* that is really declared is still
> fetched (S06). The evidence for withholding a member is weaker — it is one named member, which is
> exactly what the spec calls a minimal slice — so it is left alone rather than swept along.

That reservation was right to make and is now resolved, because the evidence turns out to be
explicit rather than weak. It was simply in a document ADR-A012 did not quote.

## What the three cases actually are

Measured on the M1 fixture, `laravel/framework` v12.64.0. The brief's warning not to conflate
`Framework::member` with `Framework facade::member` is well founded — **they have different
sources of truth**:

| Reference | Declared in the located file? | Evidence that it exists | v0.1.0 → M4 behaviour |
|---|---|---|---|
| `Log::info` | **No.** `Log.php` declares only `getFacadeAccessor()` | a `@method static void info(…)` tag at `Log.php:25`, on a `Facade` subclass | settled by M3 — no item, cited diagnostic |
| `Str::slug` | **Yes** — `public static function slug(…)` at `Str.php:1552` | the declaration itself | **fetched, 324 tokens** |
| `Arr::only` | **Yes** — `public static function only(…)` at `Collections/Arr.php:725` | the declaration itself | **fetched, 73 tokens** |

`Str` and `Arr` are not facades and carry no `@method` tags. M3's rule correctly does not fire on
them. So the question ADR-A011 answered does not reach them, and a second question has to be asked:
**may the tool fetch a dependency's real, declared method?**

## The decision

> **A `Class::member` reference whose class the PSR-4 map places outside the project's own source is
> settled when the member genuinely exists there — no bundle item, one cited diagnostic, the source
> not fetched. When the member does not exist there, nothing changes: it remains an
> `unresolved-reference` flag.**

Existence means either of the two evidences above: a real declaration the slicer can find, or a
`@method static` tag (ADR-A011's rule, which keeps working and is checked first so its richer
citation is preserved).

| Reference | Class placed | Member exists there | Outcome |
|---|---|---|---|
| `App\Services\Registry::create` | project | yes | **fetched** — unchanged |
| `App\Models\Package::active` | project | via `scope` convention | **fetched** — unchanged (ADR-A011) |
| `App\Models\Package::activatte` | project | no | **flag** — unchanged |
| `App\Facades\Pay::charge` | project | `@method static` tag | **settled** — unchanged (ADR-A011) |
| `Illuminate\Support\Facades\Log::info` | dependency | `@method static` tag | **settled**, tag citation — unchanged (ADR-A011) |
| `Illuminate\Support\Str::slug` | dependency | declared | **settled**, declaration citation — **new** |
| `Illuminate\Support\Str::slugg` | dependency | **no** | **flag** — unchanged |
| anything unplaceable | — | — | **flag** — unchanged |

The new diagnostic is
`dependency member: Illuminate\Support\Str::slug declared at vendor/…/Str.php:1543; source not fetched`.
The line is the member's **slice start** — its docblock included, the same span a fetched slice would
have had — so a reviewer opening the file lands on the whole declaration rather than mid-comment.

### The existence check is not optional

Withholding on ownership alone would silently swallow `Str::slugg()` — a typo against a real
dependency — and report it as provided. That is precisely the failure M0 records as risk **R1**
(*suppression becoming silence*) and P10 forbids: *"A missing capability should surface as an
explicit assumption in the bundle, never as a silent omission."* So a verdict is issued only on
**positive evidence that the member is there**, never on the class's ownership alone. This mirrors
ADR-A011's rule that a framework verdict needs a parsed tag, not a name match.

## Why this follows from the frozen evidence

### 1 · The fetch-collaborator move is scoped to application code, in its own title

`context-types.md`:

> ## Type 2 · Named collaborator code (**application code**, depth one)
> **What:** A class, model, enum, or **method** the diff explicitly references.
> **Seen in:** Experiment 1 (`PlaidAccount` model); Experiment 4 (`PlaidClient` method,
> `PlaidItemStatus` enum).

The type that authorises fetching a named reference names **methods** explicitly and scopes itself
to **application code** in its heading. All three cited examples are the application's own. This is
the same sentence that justified ADR-A012 for classes, and it is *more* explicit about members than
about classes.

### 2 · The taxonomy has no context type for dependency source

Six context types were derived from findings that actually occurred: same-file, named collaborator
(application code), caller, schema, configuration, runtime. Experiment 3 widened the taxonomy — its
headline is that context is *"not limited to application code"* — and what it added was **config and
schema**, not a dependency's source. Nothing in four commits asked for a package's code. ADR-A003:
*"The build takes exactly the proven moves and no others."*

### 3 · Experiment 2 says the reviewer already has it

> A naked-diff review **that knows Laravel** catches these unaided.

and the spec's precision criterion: on Experiment 2 *"the correct bundle is almost empty — pulling
the model 'just in case' is a precision failure, not caution."*

### 4 · The outcome shape already exists, and no premise may be added

ADR-A009: *"a premise exists only where an unverified premise exists."* The run located the class,
found the member, and can cite the line. Nothing is unverified, so
`ASSUMPTION: named reference could not be resolved on disk` would be **false** — the failure freeze
review 06 refused. What remains is freeze review 05's successful negative: **no item, one
diagnostic**, exactly as ADR-A011 and ADR-A012 use it.

### 5 · Why ADR-A012's own reservation no longer holds

ADR-A012 hesitated because the spec blesses *"the minimal source slice (ideally one method or class
member, not a whole file)"* — and a dependency method is one member. That sentence governs **how
large a fetched payload may be**, not **whose code may be fetched**. Type 2 governs the second
question, and it answers it. Size was never the discriminator; ownership is, for members exactly as
for classes.

## Answers to the questions the milestone posed

| Question | Answer |
|---|---|
| Should a declared dependency member produce no item? | **Yes** |
| A diagnostic? | **Yes**, one, citing file and line |
| A fetched framework slice? | **No** |
| Remain flagged? | **Only when the member is not actually there** |
| Another already-supported outcome? | It *is* one — freeze review 05's successful negative. Nothing new was introduced |
| Can the rule work without opening framework source? | The **fetch path** never opens a dependency file — the ownership guard runs before the read. The **diagnostic path** does open it, to prove the member exists and cite its line. Reading is not fetching: no dependency text reaches the bundle |
| All Composer dependencies, or only Laravel? | **All.** This is ownership, not framework knowledge. It applies identically to `Endroid\QrCode\…`, and lives on the resolver and `ClassLocator`, never behind the `FrameworkKnowledge` seam |
| Does it preserve M3? | **Yes**, and M3 remains necessary: the facade rule is checked first so `Log::info` keeps its tag citation byte-for-byte, and it is the *only* rule that covers a **project** facade, where the ownership rule does not fire at all |

## Consequences

**Positive**

- The last uncontrolled framework-source fetch in the fixture is closed: S06 in variant B goes from
  **2 fetched items / 397 tokens** to **0 items / 0 tokens** with two cited diagnostics.
- The ownership principle is now uniform. `resolve()` carries one guard for both classes and
  members instead of a special case for one of them, and the fetch path opens no dependency file.
- `B:over_fetch` reaches **0**. Every remaining fixture gap is a variant-A problem-A symptom or a
  recorded false negative.

**Negative, stated plainly**

- A reviewer who genuinely wanted a dependency method's body no longer gets it. The diagnostic names
  the file and line so they can open it, and Experiment 2 is the standing evidence that they do not
  need it in the bundle. If a scored run shows otherwise, this ADR is what to revisit.
- The diagnostic path reads dependency files. ADR-A012's stronger guarantee — *never opened* — holds
  for bare classes only, and this ADR narrows that claim honestly rather than restating it.
- Variant A is unchanged: `Illuminate\…` is still unplaceable there, so `Str::slug` still flags.
  Only the Composer-map question can reach it.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| **Leave members fetched (do nothing)** | Type 2 scopes the move to application code, and the fixture's own answer key marks it `over_fetch`. Leaving it would keep an inconsistency the evidence does not support |
| **Withhold on ownership alone, without checking the member exists** | Swallows `Str::slugg()`. P10 and M0 risk R1 forbid it, and it is the one way this rule could do real harm |
| **Fetch only a signature, not the body** | A new payload kind with no experiment behind it, and Experiment 2 says the reviewer needs neither |
| **Cap the fetched size instead** | Size was never the discriminator — ADR-A012 rejected the same idea for classes, and here the payload is already a minimal slice |
| **Extend the `FrameworkKnowledge` seam to cover it** | It is not framework knowledge. `Str` and `Arr` carry no framework marker at all; what identifies them is where Composer puts them |
| **A catalogue of known framework methods** | Explicitly excluded by the milestone, and M0 §4.5 measured that such a catalogue is already wrong between two installed Laravel versions |

## Related

- [ADR-A012](ADR-A012-surface-move-is-for-project-classes.md) — the class half of the same rule, and
  the reservation this ADR resolves.
- [ADR-A011](ADR-A011-framework-known-recognition.md) — the facade rule, preserved and still needed.
- [ADR-A009](ADR-A009-premise-catalogue.md) — why no premise was added.
- [ADR-A005](ADR-A005-slices-not-files.md), [ADR-A003](ADR-A003-closed-move-set.md), [ADR-A010](ADR-A010-inheritance-and-annotation-traversal.md).
- The M1 fixture, scenarios **S01** and **S06**.
