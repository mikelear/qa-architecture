# Mermaid diagrams — QA architecture flows

Visual reference for the leartech QA architecture. Four diagrams covering PR-time flow, promotion + gate flow, HAR pipeline, and Renovate. Each rendered with consistent styling so component types are visually distinguishable.

## Legend

```mermaid
flowchart LR
    classDef repo fill:#e2e3e5,stroke:#383d41,stroke-width:2px
    classDef standalone fill:#d4edda,stroke:#155724,stroke-width:2px
    classDef tektonTask fill:#cce5ff,stroke:#004085,stroke-width:2px
    classDef crd fill:#d1ecf1,stroke:#0c5460,stroke-width:2px,stroke-dasharray:5
    classDef trainedModel fill:#fff3cd,stroke:#856404,stroke-width:3px
    classDef storage fill:#f8d7da,stroke:#721c24,stroke-width:2px
    classDef pr fill:#ffeeba,stroke:#856404,stroke-width:2px

    R[Repo]:::repo
    S[Standalone Service]:::standalone
    T[Tekton Task]:::tektonTask
    C[CRD / K8s resource]:::crd
    M[Uses trained model]:::trainedModel
    D[(Storage)]:::storage
    P([PR / GitHub event]):::pr
```

---

## Diagram 1 — PR-time flow on a service repo

The gate-keeping moment. Everything below fires when a developer opens a PR against, say, `leartech-broker-ui`.

```mermaid
flowchart TB
    %% PR-time flow on service repo
    %% The gate-keeping moment — coverage caught BEFORE merge

    subgraph Origin
        Dev([Developer opens PR<br/>on leartech-broker-ui]):::pr
    end

    subgraph LighthouseTriggers[Lighthouse presubmits fire — CRDs in jx ns]
        TC1[TriggerConfig:<br/>pr / unit-tests / lint]:::crd
        TC2[TriggerConfig:<br/>full-qa - dep-update label]:::crd
    end

    subgraph PreviewEnv[Preview env spin-up]
        PrevK[K8s namespace:<br/>jx-leartech-broker-ui-pr-N]:::crd
        PrevDeploy[Service deploy<br/>+ dependencies]:::crd
    end

    subgraph TestPacks[Test packs run as Tekton tasks]
        T1[end2end-ui<br/>Playwright Firefox]:::tektonTask
        T2[contract<br/>OpenAPI conformance]:::tektonTask
        T3[integration<br/>service-to-service]:::tektonTask
        T4[unit<br/>per-language]:::tektonTask
        T5[accessibility<br/>axe-core in Playwright]:::tektonTask
    end

    subgraph Analysis[Analysis Tekton tasks]
        A1[AST diff analyzer<br/>Go: go-callvis<br/>TS: ts-morph]:::tektonTask
        A2[risk-assessor<br/>combines AST + Tempo + HAR]:::tektonTask
        A3[coverage-scanner<br/>detects new test surface]:::tektonTask
        A4[ai-code-reviewer<br/>cites repo-type-policy]:::tektonTask
    end

    subgraph TrainedModel[ML / AI services — already running]
        AIClass[leartech-ai-classifier<br/>FastAPI + PyTorch<br/>code_classifier.pt]:::trainedModel
        Claude[Claude / DeepSeek / Ollama<br/>via existing AI review infra]:::trainedModel
    end

    subgraph Stores[Storage — multi-cluster aware]
        GCSResults[(GCS: results/v1/<br/>repo/SHA/cluster/<br/>SHA-keyed verdicts)]:::storage
        GCSArtifacts[(GCS: repo/pr-N/<br/>cluster/timestamp/<br/>PR-keyed artifacts)]:::storage
        GCSHAR[(GCS: har/v1/<br/>repo/SHA/run/<br/>HAR for replay)]:::storage
        Tempo[(Tempo<br/>OTel spans<br/>jx-observability)]:::storage
    end

    subgraph QAMgmt[leartech-qa-management - Git, single source of truth]
        QA1[required-tests/]:::repo
        QA2[repo-type-policy/<br/>+ risk_modifiers]:::repo
        QA3[service-catalog.yaml]:::repo
        QA4[risk-config.yaml]:::repo
    end

    subgraph CompanionPath[Companion PR auto-flow]
        CP([Companion PR opened<br/>to leartech-qa-management]):::pr
        CPCi[CI: schema validate<br/>cross-reference linter]:::tektonTask
        CPMerge{added-only?}
        TideMerge[Tide auto-merge<br/>on source PR merge]:::crd
    end

    %% Connections — Triggers fire tasks
    Dev --> TC1
    Dev --> TC2
    TC1 --> PrevK
    PrevK --> PrevDeploy
    PrevDeploy --> T1
    PrevDeploy --> T2
    PrevDeploy --> T3
    Dev --> T4
    Dev --> T5
    Dev --> A1

    %% Test packs feed stores
    T1 -->|"results.json - SHA-keyed - NEW"| GCSResults
    T1 -->|"trace.zip + .png + .webm - artifacts"| GCSArtifacts
    T1 -->|"trace.har - explicit - NEW"| GCSHAR
    T1 -.->|OTel spans during run| Tempo
    T2 --> GCSResults
    T3 --> GCSResults
    T5 --> GCSResults

    %% Analysis pipeline — risk-assessor as the hub
    A1 -->|AST predicted edges| A2
    Tempo -->|preview spans| A2
    GCSHAR -->|HAR endpoints| A2
    QA3 -->|service ownership map| A2
    QA4 -->|thresholds| A2
    A2 -->|"risk verdict + rationale"| GCSResults

    %% AI reviewer reads everything
    GCSResults --> A4
    A2 -.->|risk_level + factors| A4
    A4 -->|prompts| AIClass
    A4 -->|prompts| Claude
    A4 -->|"PR comment - cites risk"| Dev

    %% Coverage scanner
    A4 -.->|"detects new test surface"| A3
    A3 -->|prompts| AIClass
    A3 -->|opens| CP

    %% Companion PR
    CP --> CPCi
    QA1 -.->|reads current state| CPCi
    QA2 -.->|reads current state| CPCi
    CPCi --> CPMerge
    CPMerge -->|yes - auto-mergeable| TideMerge
    CPMerge -->|no - removals/renames| Dev
    TideMerge -->|on source PR merge| QA1

    classDef repo fill:#e2e3e5,stroke:#383d41,stroke-width:2px
    classDef standalone fill:#d4edda,stroke:#155724,stroke-width:2px
    classDef tektonTask fill:#cce5ff,stroke:#004085,stroke-width:2px
    classDef crd fill:#d1ecf1,stroke:#0c5460,stroke-width:2px,stroke-dasharray:5
    classDef trainedModel fill:#fff3cd,stroke:#856404,stroke-width:3px
    classDef storage fill:#f8d7da,stroke:#721c24,stroke-width:2px
    classDef pr fill:#ffeeba,stroke:#856404,stroke-width:2px
```

### Notes on Diagram 1

- **`risk-assessor` is the hub** of the analysis subgraph — it combines AST (env-independent primary), Tempo spans, HAR endpoints, and the service-catalog into a risk verdict. The verdict drives both `ai-code-reviewer`'s PR comment and `coverage-scanner`'s companion-PR proposals.
- **Trained models** (yellow) appear three times in this flow — `ai-code-reviewer` calls Claude/DeepSeek for code commentary, `coverage-scanner` calls the leartech-ai-classifier for "what test should be added", and `risk-assessor` is currently rule-based but could call the classifier in v2.
- **The companion PR** is the structural fix for mqube's drift problem. It auto-merges only when the diff is `added-only`; removals/renames need human review.

---

## Diagram 2 — Promotion + gate flow (PR against GitOps repo)

What happens after the service merges. The gate sits here — `leartech-gate` is the porcupine-equivalent.

```mermaid
flowchart TB
    %% Promotion flow — PR against GitOps repo, gate fires
    %% This is mqube's "porcupine-defender" equivalent

    subgraph SourceMerge[Service repo side]
        SrcMerge([Service PR merges]):::pr
        ReleasePipe[Tekton: release pipeline<br/>build + scan + push image]:::tektonTask
        ReleasePipe2[Tekton: promote-helm-release<br/>opens helmfile PR]:::tektonTask
    end

    subgraph GitOpsRepo[JX3_Leartech_Production - the GitOps repo]
        GOPRepo[GitOps repo<br/>helmfiles/leartech-prod/]:::repo
        PromoPR([Promotion PR<br/>updates service version<br/>opened as DRAFT or READY<br/>per autoMerge config]):::pr
    end

    subgraph LighthouseGate[Lighthouse presubmits on promotion PR]
        TG1[TriggerConfig: verify<br/>helmfile validation]:::crd
        TG2[TriggerConfig: lint-overrides<br/>configs/ paths]:::crd
        TG3[TriggerConfig: leartech-gate<br/>always_run, optional false]:::crd
        TG4[TriggerConfig: beaver-notes<br/>release-notes generator - if adopted]:::crd
        TG5[TriggerConfig: check-new-releases<br/>helmfile drift detector - if adopted]:::crd
    end

    subgraph GateCore[leartech-gate - the QA gate]
        GBin[leartech-gate binary<br/>Go - data-driven quills]:::standalone
        GHelm[helmfile diff via beaver lib<br/>identifies promoted services]:::tektonTask
        Q1[shift-left-tests quill<br/>required tests passed?]:::tektonTask
        Q2[contract-tests quill]:::tektonTask
        Q3[co-promotion quill<br/>service pairs together?]:::tektonTask
        Q4[migrations quill]:::tektonTask
        Q5[load-sla quill<br/>Phase 2]:::tektonTask
        QRisk[risk-override quill<br/>Phase 2.5<br/>reads risk verdict]:::tektonTask
    end

    subgraph QAMgmt2[leartech-qa-management]
        QM1[required-tests/]:::repo
        QM2[gate-metadata/quills.yaml]:::repo
        QM3[load/service.yaml<br/>SLAs]:::repo
        QM4[overrides.log.jsonl]:::repo
    end

    subgraph Stores2[Storage]
        GCSR[(GCS: results/v1/<br/>SHA-keyed)]:::storage
        GCSRisk[(GCS: results/v1/risk/<br/>risk verdict)]:::storage
    end

    subgraph Override[Override path]
        FailCheck{Any quill fail?}
        OverrideCmd([/override leartech-gate<br/>via Lighthouse plugin]):::pr
        TwoKey{High-risk?<br/>two-key required?}
        Approver([/approve override<br/>different actor]):::pr
        AuditLog[(Append to<br/>overrides.log.jsonl)]:::storage
    end

    subgraph Merge[Final merge]
        Tide2[Tide auto-merge<br/>Lighthouse]:::crd
        BootJob[K8s Job: bootjob<br/>helmfile sync to cluster]:::crd
        ProdDeploy[New ReplicaSet<br/>in leartech-prod]:::crd
    end

    %% Service merge → promotion PR
    SrcMerge --> ReleasePipe
    ReleasePipe --> ReleasePipe2
    ReleasePipe2 -->|opens promotion PR| GOPRepo
    GOPRepo --> PromoPR

    %% Lighthouse fires checks
    PromoPR --> TG1
    PromoPR --> TG2
    PromoPR --> TG3
    PromoPR --> TG4
    PromoPR --> TG5

    %% Gate runs
    TG3 --> GBin
    GBin --> GHelm
    GHelm -->|"per service: name+version"| Q1
    GHelm --> Q2
    GHelm --> Q3
    GHelm --> Q4
    GHelm --> Q5
    GHelm --> QRisk

    %% Quills read qa-management + result store
    QM1 -->|"required test names"| Q1
    QM2 -->|"quill config"| Q1
    QM2 -->|"quill config"| Q2
    QM2 -->|"quill config"| Q3
    QM2 -->|"quill config"| Q4
    QM3 -->|"SLA thresholds"| Q5
    GCSR -->|"verdicts by SHA"| Q1
    GCSR -->|"verdicts by SHA"| Q2
    GCSR -->|"verdicts by SHA"| Q4
    GCSR -->|"verdicts by SHA"| Q5
    GCSRisk -->|"risk_level high?"| QRisk

    %% Decision
    Q1 --> FailCheck
    Q2 --> FailCheck
    Q3 --> FailCheck
    Q4 --> FailCheck
    Q5 --> FailCheck

    FailCheck -->|all pass| Tide2
    FailCheck -->|any fail| OverrideCmd

    %% Override path — risk-driven stiffening
    OverrideCmd --> TwoKey
    QRisk -->|risk_level - approver_required| TwoKey
    TwoKey -->|"yes - need second approver"| Approver
    TwoKey -->|"no - single override"| Tide2
    Approver --> Tide2
    OverrideCmd -.->|webhook| AuditLog
    Approver -.->|webhook| AuditLog
    AuditLog -.->|scheduled scan| QM4

    %% Tide → bootjob → deploy
    Tide2 --> BootJob
    BootJob --> ProdDeploy

    classDef repo fill:#e2e3e5,stroke:#383d41,stroke-width:2px
    classDef standalone fill:#d4edda,stroke:#155724,stroke-width:2px
    classDef tektonTask fill:#cce5ff,stroke:#004085,stroke-width:2px
    classDef crd fill:#d1ecf1,stroke:#0c5460,stroke-width:2px,stroke-dasharray:5
    classDef trainedModel fill:#fff3cd,stroke:#856404,stroke-width:3px
    classDef storage fill:#f8d7da,stroke:#721c24,stroke-width:2px
    classDef pr fill:#ffeeba,stroke:#856404,stroke-width:2px
```

### Notes on Diagram 2

- **The gate itself is a Go binary** running inside a Tekton task — not a standalone deployment. It executes, exits, posts a check status. Not always-on.
- **Risk-driven override stiffening** — when `risk_level: high`, the gate reads `override_policy.two_key_required: true` and `/override leartech-gate` requires a second `/approve override` from a different actor. Mqube has flat overrides; ours scale with risk.
- **The audit log** is appended to via webhook (Lighthouse → small webhook receiver → PR or scheduled scan). Worth ironing out: how exactly does the webhook write find its way back to `gate-metadata/overrides.log.jsonl` without contention?

---

## Diagram 3 — HAR pipeline + load testing (multi-producer)

The strategic infrastructure. Multiple producers feed one engine; load testing is the first consumer but more come.

```mermaid
flowchart TB
    %% HAR pipeline — multi-producer interchange format
    %% Strategic infra; load testing is one of several consumers

    subgraph Producers[HAR producers - layered over phases]
        P1[Phase 1<br/>Playwright HAR<br/>from end2end-ui task]:::tektonTask
        P2[Phase 2<br/>tempo-to-har<br/>Go service - standalone]:::standalone
        P3[Phase 2 optional<br/>gateway log to HAR<br/>if a gateway is adopted]:::tektonTask
        P4[Phase 2 optional<br/>OpenAPI synth to HAR<br/>schema-derived stub]:::tektonTask
        P5[Phase 3 conditional<br/>Pixie or Hubble exporter<br/>eBPF flows to HAR]:::standalone
    end

    subgraph TempoSource[Tempo - already running per golden-service-standard]
        TempoStore[(Tempo<br/>OTel spans<br/>jx-observability)]:::storage
        OTelInstr[Every leartech Go service<br/>emits OTel by mandate]
    end

    subgraph SanitizeLayer[Sanitize/template layer - shared library]
        SLib[leartech-go-common/sanitize<br/>strip auth headers - PII patterns<br/>parameterize - case-id - user-id]:::tektonTask
        ReplayAuth[Replay-time auth substitution<br/>OAuth client-credentials<br/>fresh token per replay]:::tektonTask
    end

    subgraph HARStorage[HAR storage]
        HARGCS[(GCS: har/v1/<br/>repo/SHA/run-id/<br/>multiple sources<br/>indexed)]:::storage
    end

    subgraph LoadEngine[leartech-load-testing - HAR replay engine]
        LTSvc[leartech-load-testing<br/>Go service<br/>Standalone Deployment :8080<br/>API: replay, results, blueprints]:::standalone
        LTBlueprints[Blueprints handler<br/>fetches HAR by ID]:::tektonTask
        LTReplay[Replay handler<br/>filters entries<br/>fires concurrent HTTP<br/>OAuth-substituted]:::tektonTask
        LTSLA[SLA assertion engine<br/>p95 - error_rate<br/>vs baseline]:::tektonTask
    end

    subgraph Triggers[Replay triggers]
        Cron1[K8s CronJob: nightly<br/>per-service load test<br/>generated from load configs]:::crd
        Cron2[Tekton step in gate<br/>perf-critical services<br/>inline replay during promotion]:::tektonTask
    end

    subgraph QAMgmt3[leartech-qa-management]
        QM_load[load/service.yaml<br/>SLA + filter config]:::repo
    end

    subgraph Stores3[Storage]
        GCSR3[(GCS: results/v1/<br/>load-test verdicts<br/>SHA-keyed)]:::storage
        TempoOut[(Tempo<br/>load-test spans<br/>full distributed trace)]:::storage
    end

    subgraph Consumers[HAR consumers - many, cheap to add]
        C1[1 - Load testing<br/>Phase 2 - first consumer]
        C2[2 - Risk-assessor<br/>Phase 2.5<br/>HAR endpoint extraction<br/>as confirming signal]
        C3[3 - Test-generation AI<br/>Phase 3<br/>diff: prod traffic vs tested<br/>propose Playwright stubs]:::trainedModel
        C4[4 - Contract derivation<br/>Phase 3<br/>HAR pairs to Pact contracts]
        C5[5 - Security replay<br/>Phase 3<br/>HAR + mutations<br/>basic DAST]
        C6[6 - Coverage gap reporter<br/>Phase 3<br/>HAR-source diff]
    end

    %% Producer flows
    OTelInstr --> TempoStore
    TempoStore -->|span query| P2
    P1 --> SLib
    P2 --> SLib
    P3 --> SLib
    P4 --> SLib
    P5 --> SLib
    SLib -->|"sanitized HAR"| HARGCS

    %% Replay engine flow
    HARGCS --> LTBlueprints
    LTBlueprints --> LTReplay
    QM_load -->|"filter + SLA"| LTReplay
    LTReplay --> ReplayAuth
    ReplayAuth -->|"fresh token injected"| LTReplay
    LTReplay --> LTSLA
    QM_load -->|"baseline SLA"| LTSLA

    %% Triggers
    Cron1 -.->|"POST /api/loadtest/run-latest"| LTSvc
    Cron2 -.->|"POST /api/loadtest/run"| LTSvc
    LTSvc --> LTBlueprints

    %% Output
    LTSLA -->|"verdict + SLA pass/fail"| GCSR3
    LTReplay -.->|"OTel spans during replay"| TempoOut

    %% Consumer fan-out
    HARGCS --> C1
    HARGCS --> C2
    HARGCS --> C3
    HARGCS --> C4
    HARGCS --> C5
    HARGCS --> C6

    classDef repo fill:#e2e3e5,stroke:#383d41,stroke-width:2px
    classDef standalone fill:#d4edda,stroke:#155724,stroke-width:2px
    classDef tektonTask fill:#cce5ff,stroke:#004085,stroke-width:2px
    classDef crd fill:#d1ecf1,stroke:#0c5460,stroke-width:2px,stroke-dasharray:5
    classDef trainedModel fill:#fff3cd,stroke:#856404,stroke-width:3px
    classDef storage fill:#f8d7da,stroke:#721c24,stroke-width:2px
    classDef pr fill:#ffeeba,stroke:#856404,stroke-width:2px
```

### Notes on Diagram 3

- **Producers are layered by phase** — Playwright (already free, Phase 1), Tempo→HAR (Phase 2, reuses existing observability), gateway-log + OpenAPI synth (Phase 2 optionals), Pixie/Hubble (Phase 3 conditional). Each addition is independent.
- **Sanitize/template is two phases** — storage-time redaction (strips auth/PII so HAR is safe to keep) + replay-time auth substitution (mints fresh OAuth tokens). Both phases reuse `leartech-go-common`.
- **The replay engine is a standalone Go service**, lifted from `mqube-load-testing`. Has its own HTTP API (port 8080) so triggers can come from CronJob curl, Tekton step, or manual ops calls.
- **Six consumers shown** — only #1 (load testing) is Phase 2; the rest are Phase 3 cheap additions because the HAR pipeline already exists. **#3 test-generation AI** is the one that closes the loop — Tempo shows production traffic, Playwright HARs show what's tested, the diff becomes a PR proposing new Playwright tests.

---

## Diagram 4 — Renovate hardening (cross-cutting)

Renovate is its own thread that touches all three flows above.

```mermaid
flowchart LR
    %% Renovate hardening — touches all three flows
    RBot[Renovate self-hosted<br/>Mon 7am UTC<br/>scans all leartech repos]:::standalone
    DepPR([Dep-update PRs<br/>across services]):::pr
    LblA[Label: dep-update]
    LblB[Label: load-bearing-dep]
    FullQA[Tekton trigger: full-qa<br/>full Playwright suite<br/>against preview env]:::tektonTask
    HumanGate[human-review-required<br/>pseudo-check<br/>always failing<br/>until /override]:::crd
    StdFlow[Standard PR-time flow<br/>see Diagram 1]
    
    RBot -->|opens| DepPR
    DepPR --> LblA
    DepPR --> LblB
    LblA --> FullQA
    LblB --> HumanGate
    FullQA --> StdFlow
    HumanGate -->|CODEOWNERS approval| StdFlow

    classDef repo fill:#e2e3e5,stroke:#383d41,stroke-width:2px
    classDef standalone fill:#d4edda,stroke:#155724,stroke-width:2px
    classDef tektonTask fill:#cce5ff,stroke:#004085,stroke-width:2px
    classDef crd fill:#d1ecf1,stroke:#0c5460,stroke-width:2px,stroke-dasharray:5
    classDef pr fill:#ffeeba,stroke:#856404,stroke-width:2px
```

### Notes on Diagram 4

- **Renovate is already self-hosted on GCP** (per `~/leartech/hub/status/code-standards.md` line 21). What's new is the `full-qa` trigger and the load-bearing-dep policy.
- **Two parallel labels** — `dep-update` triggers the full-qa pack; `load-bearing-dep` blocks via `human-review-required`. Both can be true on a single PR.
- **CODEOWNERS sign-off** is the override path for load-bearing changes; not flat `/override` because the risk is high enough that we want named owners involved.

---

## Component-type summary

For asking follow-ups about deployment shape:

| Component | Type | Where it runs |
|---|---|---|
| `leartech-qa-management` | **Repo** | GitHub; consumed by everyone via Renovate-pinned tags |
| `leartech-gate` | **Go binary in Tekton task** | Runs as PipelineRun on every promotion PR; not always-on |
| `leartech-load-testing` | **Standalone Service** | Long-running Deployment in `leartech-loadtest` ns; HTTP API on :8080 |
| `tempo-to-har` | **Standalone Service** (Phase 2) | Long-running Deployment; queries Tempo on schedule, pushes HAR to GCS |
| `risk-assessor` | **Tekton task** (Phase 2.5) | Runs per-PR; combines AST output + Tempo + HAR; not a standalone svc |
| `coverage-scanner` | **Tekton task** (Phase 2) | Runs per-PR; reads service-catalog + diff; opens companion PR via GitHub API |
| `ai-code-reviewer` | **Tekton task** (already exists) | Calls existing leartech-ai-classifier + Claude/DeepSeek |
| Lighthouse `TriggerConfig` / `Plugins` | **CRDs** | In `jx` namespace; config-driven; control which presubmits fire |
| `K8s CronJob` (nightly load test) | **CRD** | One per service in `jx-staging` |
| Tekton `Task` / `PipelineRun` | **CRDs** | Ephemeral; created per run, garbage-collected |
| GCS buckets | **External storage** | `test-artifacts-product-first` (existing); paths added: `results/`, `har/` |
| Tempo + Loki + Prometheus | **External infra** | `jx-observability`; already running |
| Renovate | **Standalone service** | Already running on GCP, scans Mon 7am |
| `leartech-ai-classifier` | **Standalone Service** (already running) | jx-staging GCP; FastAPI + PyTorch model |

---

## Five things worth digging into

Anchor points for follow-up questions:

1. **The risk-override flow** — how exactly does the gate know to require two-key on high-risk PRs? It's via the `risk-override quill` (Diagram 2) which reads the risk verdict from GCS and modifies `override_policy.two_key_required` dynamically.

2. **Companion-PR auto-merge dependency** — how does Tide know to merge the qa-management companion PR only when the source service PR merges? Lighthouse supports cross-repo PR dependencies via labels (`depends-on:leartech-broker-ui#4521`), but the implementation detail is worth tightening.

3. **Producer-to-consumer fan-out for HAR** (Diagram 3) — the diff between #1 (load testing today) and #3 (test-generation tomorrow) is just a different consumer; nothing producer-side changes. Worth confirming the HAR storage path schema is consumer-agnostic.

4. **Trained-model usage** appears in three boxes (yellow). Worth deciding whether risk-assessor stays rule-based or learns from override patterns over time — Q22 in `open-questions.md`.

5. **Tempo as both source and sink** (Diagram 3) — load-testing both reads Tempo (via tempo-to-har) and writes Tempo (during replay). Could create a feedback loop if not careful — load-test traffic gets re-ingested as "real traffic" for next replay generation. Worth a guard.

---

## How to view these locally

GitHub renders Mermaid natively in `.md` files. So does most modern markdown previewers (VS Code, JetBrains, etc.). For ad-hoc rendering: paste a single diagram into <https://mermaid.live/> to iterate visually.

To export as PNG for slide decks: `mermaid-cli` (`npm install -g @mermaid-js/mermaid-cli`), then `mmdc -i diagrams.md -o out.png`.
