# 04 · Diagrams

Mermaid source, so the diagrams live in version control and diff. Four diagrams, exactly as
required: high-level architecture, request flow, Context Discovery pipeline, module dependency.

## 1 · High-level architecture

```mermaid
flowchart LR
  subgraph OUT1["Outside · inputs (read-only, local)"]
    D["Unified diff<br/>file or stdin"]
    R["Target repository<br/>PHP / Laravel source"]
    CJ["composer.json<br/>PSR-4 autoload map"]
    B["--budget (tokens)"]
  end

  subgraph SYS["context-discover · one process, one command"]
    CLI["Cli · options, wiring, exit codes"]
    PIPE["Pipeline · DiscoverContext (9 stages)"]
    DISC["Discovery · extraction / lever / resolution / flagging"]
    ASM["Assembly · bundle, priority, budget"]
    AD["Adapters · diff, filesystem, PSR-4, tokenizer, grep, writers"]
    CLI --> PIPE --> DISC
    PIPE --> ASM
    DISC -. via Ports .-> AD
    ASM -. via Ports .-> AD
  end

  subgraph OUT2["Outside · outputs"]
    JB["Context bundle · JSON (bundle_version 1)"]
    MB["Context bundle · Markdown"]
    DIAG["Diagnostics · stderr"]
  end

  subgraph DOWN["Downstream · NOT in scope"]
    HUM["Human assembles prompt: diff + bundle"]
    LLM["Reviewer model (condition C)"]
    SCORE["A-vs-C scoring against Phase 0 keys"]
  end

  D --> CLI
  R --> AD
  CJ --> AD
  B --> CLI
  SYS --> JB
  SYS --> MB
  SYS --> DIAG
  JB --> HUM --> LLM --> SCORE

  classDef excl stroke-dasharray: 5 4;
  class DOWN,HUM,LLM,SCORE excl;
```

The dashed region is deliberately outside the boundary: no LLM call inside the tool (X5), no
judgement (P6, X4), no scoring (Phase 2).

## 2 · Request flow (one invocation)

```mermaid
sequenceDiagram
  autonumber
  actor U as Operator
  participant CLI as Cli\DiscoverCommand
  participant P as Pipeline\DiscoverContext
  participant DP as UnifiedDiffParser
  participant SR as LocalSourceRepository
  participant AX as AssertionExtractor
  participant LP as LeverPolicy
  participant RS as Resolvers (own-file / named-ref / caller)
  participant AW as AssumptionWriter
  participant BA as BundleAssembler
  participant BE as BudgetEnforcer
  participant W as BundleWriter

  U->>CLI: context-discover --diff --repo --budget
  CLI->>CLI: validate options (budget required)
  CLI->>P: run(diffText, repoRoot, budget, callerScope)
  P->>DP: parse(diffText)
  DP-->>P: Diff (files, regions, changed members)
  P->>SR: text(path) for each changed file
  SR-->>P: full current file text
  P->>AX: extract(Diff, source)
  AX-->>P: Assertion[] (4 kinds, deterministic order)
  loop per assertion
    P->>LP: leverFor(assertion)
    LP-->>P: Fetched | Flagged
    alt Fetched
      P->>RS: resolve(assertion)
      RS-->>P: SourceSlice[] (empty ⇒ fall through)
      opt empty result / unreadable / unresolved
        P->>AW: statementFor(assertion)
        AW-->>P: "ASSUMPTION: …"  (P10 fail-to-flag)
      end
    else Flagged
      P->>AW: statementFor(assertion)
      AW-->>P: "ASSUMPTION: …"
    end
  end
  P->>BA: assemble(resolved, budget)
  BA-->>P: Bundle (every item has reason + lever)
  P->>BE: enforce(bundle)
  BE-->>P: Bundle with used_tokens + dropped[]
  P->>W: write(bundle)
  W-->>CLI: JSON | Markdown
  CLI-->>U: stdout (+ stderr diagnostics), exit 0
```

No step calls the network, spawns a process, writes into the repository, or loops back to an
earlier step.

## 3 · Context Discovery pipeline (the nine responsibilities)

```mermaid
flowchart TD
  S1["1 · Parse diff<br/>files · regions · changed member signatures"]
  S2["2 · Load full changed-file text<br/>(analysis input, not payload)"]
  S3{"3 · Extract assertions<br/>what the diff claims but cannot prove"}

  E1["OwnFile<br/>SameFileSymbolAbsence · sibling NamedReference<br/>R1 · Exp 1"]
  E2["NamedReference<br/>class / model / enum / method named in the region<br/>R2 · Exp 1,4"]
  E3["ChangedSignature<br/>arity or parameter shape changed<br/>R3 · Exp 1,4"]
  E4["UnverifiablePremise<br/>closed catalogue: transaction · atomic-lock-store ·<br/>schema-index-support · data-state<br/>R5 · Exp 1,3"]

  S4{"4 · LeverPolicy<br/>named + single + depth-one on disk?"}
  F1["5a · FETCH minimal slice"]
  F2["5b · FLAG one assumption sentence"]

  R1["OwnFileResolver<br/>use block · enclosing member · named siblings"]
  R2["NamedReferenceResolver<br/>PSR-4 lookup → one member / model surface / enum"]
  R3["CallerResolver<br/>bounded grep of call sites under --caller-scope"]

  S6["6 · Assemble bundle<br/>payload + reason + lever + provenance"]
  S7["7 · Estimate tokens · enforce budget<br/>drop lowest priority, record every drop"]
  S8["8 · Serialise · JSON | Markdown"]

  X1["✗ forward import-following (disproven, Exp 1)"]
  X2["✗ depth > 1 / transitive"]
  X3["✗ config / migration resolver (n=1 → flag)"]

  S1 --> S2 --> S3
  S3 --> E1 & E2 & E3 & E4
  E1 & E2 & E3 & E4 --> S4
  S4 -->|cheap, named, depth-one| F1
  S4 -->|reverse-graph-deep · runtime · data-state| F2
  F1 --> R1 & R2 & R3
  R1 & R2 & R3 --> S6
  R2 -.->|unresolved| F2
  R3 -.->|over max-call-sites| F2
  F2 --> S6
  S6 --> S7 --> S8

  S3 -.-x X1
  R2 -.-x X2
  E4 -.-x X3

  classDef banned stroke-dasharray: 4 4;
  class X1,X2,X3 banned;
```

The crossed nodes are the exclusions, drawn so the diagram shows *where* each one would have
attached — and does not.

## 4 · Module dependency

```mermaid
flowchart BT
  subgraph L0["Domain · depends on nothing"]
    DD["Domain\Diff"]
    DA["Domain\Assertion"]
    DB["Domain\Bundle"]
    DS["Domain\Source"]
  end

  subgraph L1["Ports · interfaces only"]
    PT["DiffParser · SourceRepository · ClassLocator<br/>MemberSlicer · CallSiteSearch<br/>TokenEstimator · BundleWriter"]
  end

  subgraph L2["Discovery + Assembly · pure logic"]
    EX["Discovery\Extraction (4 extractors)"]
    LV["Discovery\Lever (policy · premise catalogue)"]
    RE["Discovery\Resolution (3 resolvers)"]
    FL["Discovery\Flagging"]
    AS["Assembly (assembler · priority · budget)"]
  end

  subgraph L3["Pipeline"]
    PI["Pipeline\DiscoverContext"]
  end

  subgraph L4["Adapters · the only I/O"]
    AD["Diff · Filesystem · Autoload<br/>Php · Search · Estimation · Serialization"]
  end

  subgraph L5["Cli"]
    CL["Options · Wiring · DiscoverCommand · ExitCode"]
  end

  PT --> DD & DA & DB & DS
  EX --> PT
  LV --> PT
  RE --> PT
  FL --> DA
  AS --> PT
  PI --> EX & LV & RE & FL & AS
  AD --> PT
  CL --> PI
  CL --> AD
```

Arrows point at what a module depends on. Two rules make the graph acyclic and enforceable:
`Discovery`/`Assembly`/`Pipeline` never name an `Adapters` class, and `Cli\Wiring` is the only
place a concrete adapter appears.
