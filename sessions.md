# Sessions plan + parallelization + live status

The execution-tracking artifact for building this architecture. Three views:

1. **Linear session plan** — discrete, end-to-end-testable units
2. **Parallelization plan** — which sessions can fan out across multiple Claude Code sessions
3. **Live status** — running log, updated as sessions complete

Following the leartech "session 11/12" pattern from `~/leartech/hub/status/code-standards.md`.

---

## Part 1: Linear session plan

Each session aims for **3-6 hours of focused work** producing a **testable, committable deliverable**. Skip a session if its deliverable can't be defined cleanly — that's a sign the design needs another pass; update `open-questions.md` instead.

**Strategy update 2026-05-04**: starts with a single-engineer end-to-end spike (Session 0, ~10h) to prove the architecture, then Phase 1 sessions become hardening passes rather than build-from-scratch. Foundation (deployments, CI/CD, GCS, Lighthouse, golden templates, AI infrastructure) is already in place, so the spike is cheap and the architectural payoff is real. See `session-0-brief.md` for the spike's full scope, validation criteria, and anti-scope-creep checklist.

### Session 0 — End-to-end spike (~10h, single session)

**Reframed strategy as of 2026-05-04**: instead of building Phase 1 sequentially over ~3 weeks, do a focused full-stack spike in one day to prove every interface end-to-end at low fidelity, then Phase 1 sessions become hardening passes. The foundation (deployments, CI/CD, GCS, Lighthouse, golden templates) is already in place, so the cost of the spike is cheap and the architectural payoff is real — integration surprises get found in week 1, not week 6.

See `session-0-brief.md` for full pre-flight checklist, deliverables, validation criteria, and explicit anti-scope-creep list.

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **0** | End-to-end spike — prove the architecture works | One demo flow: PR on auth-ui → shift-left tests → results.json in GCS → synthetic promotion PR → leartech-gate fires → green/red check → /override works. ALL components in skeletal form. | new repos: `mikelear/leartech-qa-management`, `mikelear/leartech-gate` + extension to `leartech-pipeline-catalog` + sandbox helmfile repo | ~10h |

### Phase 1 — harden the spike (~6 short sessions, ~1-2h each)

After Session 0, Phase 1 sessions are hardening passes on top of the spike, not build-from-scratch. Each session has clearer scope because the architecture has been exercised under real conditions.

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **1.1** | Harden `leartech-qa-management` | Add CUE schemas; cross-reference linter (`make validate`); self-CI Lighthouse trigger; auto-merge policy for additions-only PRs; expand `required-tests/` to 3-4 services | `leartech-qa-management` | 2-3h |
| **1.2** | Extend result-store path to other test packs | Spike covered Playwright UI only. Add same SHA-keyed upload to other test packs — contract, integration, unit (per repo type) | `leartech-pipeline-catalog` | 1-2h |
| **1.3** | Second quill: contract-tests | Extract data-driven quill framework from spike's hardcoded shift-left-tests; add `contract-tests` quill following same impl pattern; `gate-metadata/quills.yaml` becomes the source of truth | `leartech-gate` + `leartech-qa-management` | 2-3h |
| **1.4** | Third quill: co-promotion | Pure helmfile diff check (no result-store query). Validates that linked services promote together. Different `impl` type than result-store-lookup → proves the framework handles multiple impl types. | `leartech-gate` | 1-2h |
| **1.5** | Wire to real GitOps repos | Spike used a sandbox. Now wire `leartech-gate` Lighthouse trigger to actual production GitOps repos (per cluster). Branch-protection rule. Multi-cluster (gcp + az) result-store query. | GitOps repos + `leartech-gate` | 2-3h |
| **1.6** | `repo-type.yaml` bulk seed | AI-suggested classification PR per active repo; CODEOWNERS confirm; 100% coverage. Spike covered auth-ui only. | many repos (one PR each) | 3-4h spread across ~2 sessions if needed |
| **1.7** | Phase 1 retrospective | Validation criteria checked (now post-hardening); `~/leartech/hub/shared-rules/platform-leverage.md` started with patterns landed; Phase 2 estimates recalibrated from actuals | hub | 1-2h |

**Phase 1 total** (post-spike): ~12-19 hours of focused hardening work; calendar ~1-1.5 weeks single-threaded, ~1 week with parallelization.

**Phase 1 + Session 0 total**: ~22-29 hours combined; calendar ~1.5-2 weeks single-threaded.

### Phase 2 — enrichment (~7 sessions)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.1** | HAR pipeline foundations | Playwright records HAR explicitly; upload to `har/v1/...`; sanitize utility forked from mqube + adapted | `leartech-pipeline-catalog` + new: `leartech-go-common/sanitize` | 3-4h |
| **2.2** | `leartech-load-testing` service | Go service from template; replay engine; OAuth substitution; Mongo + GCS write; API endpoints; chart | new repo: `mikelear/leartech-load-testing` | 6-8h |
| **2.3** | SLA quill + load configs | `load-sla` quill in gate (alert-only); `load/<service>.yaml` schema in qa-management; nightly CronJob generator | `leartech-gate` + `leartech-qa-management` + GitOps repos | 4-5h |
| **2.4** | `tempo-to-har` synthesizer | Go service that queries Tempo and emits HAR; chart; nightly schedule | new repo: `mikelear/tempo-to-har` (or as part of `leartech-load-testing`) | 3-4h |
| **2.5** | AI coverage scanner + companion PRs | Tekton task; cross-repo PR dependency wiring; auto-merge for additions; human review for removals | `leartech-pipeline-catalog` + integrations with `leartech-ai-classifier` | 5-7h |
| **2.6** | Renovate hardening | `full-qa` Lighthouse trigger; Renovate `packageRules` for load-bearing deps; bulk apply across repos | many repos + central Renovate config | 3-4h |
| **2.7** | Phase 2 retrospective | Override-rate analysis from result store; flake-budget tuning pass; platform-leverage register updated | hub + `leartech-qa-management` | 2h |

**Phase 2 total**: ~26-34 hours; calendar ~2 weeks parallel.

### Phase 2.5 — risk-based gating (~7 sessions)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.5.1** | `service-catalog.yaml` bootstrap | AI-suggested catalog entries for every active service; CODEOWNERS-confirmed PRs | `leartech-qa-management` | 4-5h |
| **2.5.2** | AST analyzer — Go | `go-callvis` wrapper or custom `go/ast` walker; Tekton task; outputs predicted-impact JSON | `leartech-pipeline-catalog` (or new `leartech-ast-analyzer` if it grows) | 6-8h |
| **2.5.3** | AST analyzer — TypeScript | `ts-morph` wrapper; same output schema | same | 5-6h |
| **2.5.4** | `risk-assessor` core | Tempo span collector; HAR endpoint extractor; rule-based classifier; result-store write of risk verdict | new repo or `leartech-pipeline-catalog` task | 6-8h |
| **2.5.5** | Gate integration + override stiffening | `risk-override` quill in `leartech-gate`; approval-required logic via Lighthouse webhook; audit log writer | `leartech-gate` | 3-4h |
| **2.5.6** | AI integration — risk in PR comments | Code-reviewer cites risk + factors; coverage-scanner uses risk-modifiers when proposing companion PR test additions | `leartech-pipeline-catalog` + `leartech-ai-classifier` integration | 4-5h |
| **2.5.7** | Phase 2.5 retro + threshold tuning | False-positive rate measurement on first 30 PRs; threshold tuning in `risk-config.yaml`; documentation | `leartech-qa-management` + hub | 2-3h |

**Phase 2.5 total**: ~30-39 hours; calendar ~3 weeks (some sessions hard to parallelize).

### Phase 2.7 — post-deploy regression detection + traffic forensics (~6 sessions)

Adds `leartech-arrivals-observer` (Fat-Controller-equivalent K8s watcher) + `post-deploy-tests` quill + traffic-forensics on regression. See `arrivals-observer.md`.

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.7.1** | Repo skeleton + K8s watcher | Go service from template; ReplicaSet informer working against jx-staging in dev cluster; Redis lock for dedup; arrival doc written to GCS on event; tests | new repo: `mikelear/leartech-arrivals-observer` | 4-5h |
| **2.7.2** | Test trigger + result polling | K8s Job dispatch via `end2end-ui` task pointed at staging URLs (parameterize PREVIEW_URL → STAGING_URL); result polling loop with newly-failed diff vs pre-merge | `leartech-arrivals-observer` + extension to `leartech-pipeline-catalog/tasks/end2end-ui/` | 4-5h |
| **2.7.3** | Tempo client + traffic-forensics engine | Tempo span query via HTTP API; edge graph extraction; rule-based diff (new edges, rate shifts, error spikes); forensics JSON written to GCS | `leartech-arrivals-observer` + `leartech-go-common/tempo` (extract pattern) | 5-6h |
| **2.7.4** | Slack alerter + integration | Direct Slack webhook (option 2); message template with forensics diff rendered; userMappings config from qa-management; @here fallback when author missing | `leartech-arrivals-observer` + `leartech-qa-management/notification-config.yaml` | 3-4h |
| **2.7.5** | `post-deploy-tests` quill | Quill in leartech-gate; alert-only initially; integration tests; PR-comment rendering of post-deploy results | `leartech-gate` + `leartech-qa-management/gate-metadata/quills.yaml` | 2-3h |
| **2.7.6** | Phase 2.7 retro + calibration | Forensics threshold tuning from first 30 regressions; `unpredicted_edge_rate` baselined; risk-assessor calibration loop documented | hub + `leartech-qa-management/risk-config.yaml` | 2h |

**Phase 2.7 total**: ~20-25 hours; calendar ~3 weeks for one engineer; ~2 weeks parallelized.

### Phase 3 — strategic / conditional

Each item is independent and gated on operational signal. **Don't pre-plan**; create a session when the trigger fires.

| Trigger condition | Session | Est |
|---|---|---|
| Tempo coverage gaps measurable in 2.5.7 retro | Pixie deploy + `pixie-to-har` exporter | ~1 week |
| First production-only regression slips through | Synthetic prod probes (Tekton CronJob) | ~1 week |
| Team explicitly wants Slack signal | Slack-alert layer (small Go service reading result-store) | ~3 days |
| Coverage-gap report shows persistent untested edges | Test-generation AI | ~3 days |
| Pact contract testing adopted | Contract-derivation tool | ~1 week |
| Security team wants automated fuzzing | Security-replay extension to `leartech-load-testing` | ~1-2 weeks |
| Cronjob-heavy services emerge | Cronjob health quill | ~3 days |

---

## Part 2: Parallelization plan

The bottleneck isn't Claude session count; it's **the contracts between sessions**. Once a contract is committed (a schema, an API, a file path), parallel sessions can consume it without coordinating.

**The rule**: commit the contract first, then fan out.

### Session 0 — single session, no parallelism

The spike is single-engineer, single-thread by design. The integration value comes from one mind holding the whole flow at once. Don't try to parallelize.

### Phase 1 dependency graph (post-spike)

The spike has already settled the load-bearing contracts (GCS path schema, qa-management YAML structure, gate framework, helmfile parse semantics). Phase 1 hardening can fan out widely:

```
1.1 (CUE schemas + linter on qa-management) ─┐
1.2 (result-store path → other test packs)   ├─► 1.5 (real GitOps wiring) ─► 1.7
1.3 (second quill: contract-tests)            │
1.4 (third quill: co-promotion)               │
1.6 (repo-type.yaml bulk seed) ───────────────┘
```

**All five hardening sessions (1.1, 1.2, 1.3, 1.4, 1.6) are independent of each other** because the spike already proved the framework. They can run as 4 parallel streams; 1.5 + 1.7 converge after.

| Stream | Sessions | Why parallel |
|---|---|---|
| A | 1.1 (qa-management hardening: CUE + linter + auto-merge) | Different repo |
| B | 1.2 (result-store path → other test packs) | Different repo (pipeline-catalog) |
| C | 1.3 → 1.4 (more quills in gate) | Different repo (leartech-gate) |
| D | 1.6 (repo-type.yaml bulk seed) | Different repos (per-service PRs) |
| All converge | 1.5 (real GitOps wiring) → 1.7 (retro) | Needs all hardening complete to gate real promotions |

**Parallel saving**: with 2 engineers, ~3 days off Phase 1 hardening. With 3, ~5 days. Bigger benefit than the original sequential plan because the contracts are settled.

### Phase 2 dependency graph

```
After Phase 1:
   ├─► 2.1 → 2.2 → 2.3 (HAR + load testing + SLA quill)
   ├─► 2.4 (tempo-to-har, independent)
   ├─► 2.5 (AI coverage scanner, independent)
   └─► 2.6 (Renovate hardening, mostly config)
```

**Four parallel streams** is realistic. The integration retro (2.7) merges learnings; until then each stream owns its own repo + Tekton task.

**Parallel saving**: ~1 week off Phase 2 with two engineers.

### Phase 2.5 dependency graph

```
   ├─► 2.5.1 (service-catalog bootstrap, independent)
   ├─► 2.5.2 (Go AST analyzer)
   ├─► 2.5.3 (TS AST analyzer — same pattern, different language)
   └─► 2.5.4 (risk-assessor core — stubs AST inputs initially)
   
   then converge:
   ├─► 2.5.5 (gate integration)
   └─► 2.5.6 (AI reviewer integration)
```

**Parallel saving**: ~1.5 weeks off Phase 2.5 with two engineers.

### Phase 2.7 dependency graph

```
   2.7.1 (K8s watcher skeleton) ─► 2.7.2 (test trigger) ─► 2.7.3 (forensics)
                                                          ├─► 2.7.4 (Slack alerter, parallel to forensics finalisation)
                                                          └─► 2.7.5 (gate quill, parallel)
                                                                        └─► 2.7.6 (retro)
```

**Parallel streams**:
- Stream A: 2.7.1 → 2.7.2 → 2.7.3 (critical path: watcher → trigger → forensics engine)
- Stream B: 2.7.4 (Slack alerter — can start after 2.7.2 outputs are in GCS)
- Stream C: 2.7.5 (gate quill — can start after 2.7.2 settles the result-store schema for post-deploy)

**Parallel saving**: ~1 week off Phase 2.7 with two engineers.

### Mechanics — running multiple Claude Code sessions

#### Terminal layout

- **One pane per session** in tmux / iTerm splits
- **One repo per session** — different working directories
- **Name sessions clearly** — title bar set to "qa-arch / 1.2 result-store" etc.

#### Working directories

Each session needs its own checkout. Two patterns:

1. **Different repos** — natural separation. `~/leartech/leartech-gate/` and `~/leartech/leartech-load-testing/` don't conflict.
2. **Same repo, different branches** — use `git worktree`:
   ```bash
   cd ~/leartech/leartech-gate
   git worktree add ../leartech-gate-quill-2 feature/contract-quill
   ```
   Each worktree is an independent checkout sharing the same `.git`. Two sessions can work on different branches without stepping on each other.

#### Coordination via hub

The hub becomes the bulletin board. Each session, on completing meaningful work, updates `~/leartech/hub/status/qa-architecture.md` with a one-liner. The next session that starts can `cat` the file to know ground state.

See the live-status section below for the format.

#### Per-session brief

Each Claude Code session starts with a focused brief naming:

- The session ID and goal
- Pre-requisites (other sessions that must be done first; commits / tags they should pull)
- Exact deliverable
- Files to read first
- Things NOT to do (preventing scope creep)

Example template:

```
Session 1.3: leartech-gate skeleton + first quill

Pre-reqs: Phase 1 session 1.1 done (schemas committed at qa-management
v0.1.0); leartech-go-service-template available.

Deliverable: 
- Go service in mikelear/leartech-gate
- Helmfile parse via beaver lib
- shift-left-tests quill green against synthetic helmfile + GCS results
- Tekton task wrapper in pipeline-catalog with `uses:` pattern

Read first:
- ~/leartech/qa-architecture/gate.md
- ~/leartech/qa-architecture/qa-management-repo.md (schema)
- mikelear/leartech-qa-management at v0.1.0

Don't:
- Add multiple quills (that's session 1.4)
- Wire to a real GitOps repo (that's session 1.5)
```

### Disciplines that make parallelization work

Five things matter more than session count:

1. **Contract-first commits.** Before fanning out, commit the load-bearing contract. For Phase 1, that's the qa-management schemas + result-store path. For Phase 2, the HAR storage path schema. For Phase 2.5, the risk-verdict JSON schema. **No contract → no parallelism.**

2. **Small frequent commits.** Each session commits every 30-60 min. Other sessions pull frequently. Late merges are the killer.

3. **One repo, one session.** Don't run two sessions in the same working dir. Use worktrees if you must, separate repos otherwise.

4. **Hub status as bulletin board.** Update on completion of each session; readable in 30s by another session.

5. **The human as router.** With 3 sessions running, the human triages: reads outputs, spots cross-session conflicts, redirects when sessions wander. **Don't run more than 3 simultaneously** — the triage burden grows quadratically.

### What NOT to parallelize

- **Initial repo creation** — `gh repo create` is one-and-done; do it serially.
- **Schema-defining sessions** — single source of truth means single author at a time.
- **Cross-cutting refactors** — when extracting to `leartech-go-common`, one session at a time on the shared lib.
- **The retrospective** — bring streams together, don't fan out the analysis.

### Realistic calendar with parallelism (post-spike)

```
Day 1 (one engineer, single-thread)
  Session 0 — end-to-end spike (~10h focused)
  Outcome: every interface exercised end-to-end at low fidelity
           hardening backlog written based on what felt fragile

Week 1 (post-spike, parallel hardening)
  Stream A: 1.1 (qa-management CUE + linter + auto-merge)
  Stream B: 1.2 (result-store path → other test packs)
  Stream C: 1.3 → 1.4 (more quills, framework extraction)
  Stream D: 1.6 (repo-type.yaml bulk seed; ~3 days elapsed)

Week 2
  Stream A converges: 1.5 (real GitOps wiring, multi-cluster)
                       → 1.7 (retro)

Week 3
  Stream A: 2.1 → 2.2 → 2.3 (HAR + load testing)
  Stream B: 2.4 (tempo-to-har)
  Stream C: 2.5 (AI coverage scanner)
  Stream D: 2.6 (Renovate hardening, small)

Week 4
  All streams converge → 2.7 retro

Week 5
  Stream A: 2.5.1 (catalog bootstrap)
  Stream B: 2.5.2 (Go AST)
  Stream C: 2.5.3 (TS AST)
  Stream D: 2.5.4 (risk-assessor core, stubs AST)

Week 6
  Stream A: 2.5.5 (gate integration)
  Stream B: 2.5.6 (AI reviewer integration)
  Convergence → 2.5.7 retro

Week 7
  Stream A: 2.7.1 (arrivals-observer skeleton)
  Stream B: 2.7.2 (test trigger via end2end-ui)
  Stream C: 2.7.3 (forensics engine)

Week 8
  Stream A: 2.7.4 → 2.7.5 (Slack + post-deploy quill)
  Convergence → 2.7.6 retro
```

That's **~8 weeks elapsed with two engineers including spike + Phase 1/2/2.5/2.7**; **~6.5 weeks with three engineers** at peak fan-out. Spike is single-thread; everything after fans out aggressively because contracts are settled.

The bottleneck switches from "engineering capacity" to "human triage capacity" pretty quickly. Two parallel sessions is sustainable; four is intense; six is unsustainable.

---

## Part 3: Live status

Updated as sessions complete. Format: one row per session, with a one-line note on what landed. Sessions in flight tagged `🚧`; completed `✅`; blocked `⛔`.

### Session 0 (spike)

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 0 | ✅ | **complete 2026-05-05** — full bootstrap end-to-end + four-mode demo. 4 bootstrap gaps captured + fixed (chart-dir, bare-name, Postgres-default, stale-promotion-PRs). Gate logic working locally against real GCS + qa-management. Image cluster-publish is Phase 1 hardening. See `session-0-lessons.md` for the runbook. | canary `mikelear/leartech-qa-canary@main`; sandbox `mikelear/leartech-qa-sandbox-gitops@main`; qa-management `mikelear/leartech-qa-management@main`; gate `mikelear/leartech-gate@main`; catalog `mikelear/leartech-pipeline-catalog@5029bf1` (end2end upload), `@f02de5d` (qa-gate task) |
| 0c | ✅ | **complete 2026-05-05** — cluster-side loop closed on GCP. `openapi-generation` removed from gate's release (CLI not API service); Dockerfile multi-binary built; `qa-gate` task pinned to GAR image with `command:` (distroless) + `CLUSTER_TAG` from cluster-config CM. Demoed happy + failure + override on sandbox PRs. AZ cluster cross-cloud GAR auth deferred to Phase 1 hardening. 2 new bootstrap gaps captured (#5 openapi-generation for non-API services; #6 multi-binary Dockerfile). | gate `mikelear/leartech-gate@abb32ef` (Dockerfile fix); catalog `mikelear/leartech-pipeline-catalog` (qa-gate task pointing at GAR); sandbox demos in PRs #1 and #2 (closed) on `mikelear/leartech-qa-sandbox-gitops` |
| 2.4 | 🚧 | **partial 2026-05-05** — `tempo-to-har` repo bootstrapped from golden template using corrected runbook (zero residuals on first try — validates the runbook). Tempo client + OTLP→HAR converter + GCS uploader written; `/synth` CLI binary builds + ships to `us-central1-docker.pkg.dev/product-first/oci/tempo-to-har:0.0.1`. Debug Pod can pull image + run binary against in-cluster Tempo URL. **BLOCKED on live validation: Tempo backend not deployed on either cluster** — only Prometheus exists in `jx-observability`. Tempo install becomes a precursor session before tempo-to-har can produce real HARs. Bootstrap gap #7 captured: CamelCase variant (`GoServiceTemplate` → `TempoToHar`) needs sed substitution beyond kebab-case. | repo `mikelear/tempo-to-har@69b9aac`; image `tempo-to-har:0.0.1` in GAR; source-config registered in both `jx-build-cluster-{gsm,akv}` |

### Phase 1 (post-spike hardening)

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 1.1 | ⏳ | hardening — CUE schemas + linter + auto-merge on qa-management | — |
| 1.2 | ⏳ | hardening — extend result-store path to other test packs | — |
| 1.3 | ⏳ | hardening — second quill (contract-tests); extract framework | — |
| 1.4 | ⏳ | hardening — third quill (co-promotion); proves framework handles multiple impls | — |
| 1.5 | ⏳ | hardening — wire to real GitOps repos (was sandbox in spike); multi-cluster | — |
| 1.6 | ⏳ | hardening — bulk repo-type.yaml seed (was just auth-ui in spike) | — |
| 1.7 | ⏳ | retro + Phase 2 estimate recalibration + platform-leverage register seed | — |

### Phase 2

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 2.1 | ⏳ | not started | — |
| 2.2 | ⏳ | not started | — |
| 2.3 | ⏳ | not started | — |
| 2.4 | ✅ | **complete 2026-05-05** — Tempo install (Platform Foundations) unblocked validation. canary OTLP exporter wired + emitting; spans visible in Tempo (`/api/search?tags=service.name=leartech-qa-canary` returned 50 traces); `make debug-synth-canary` against real spans returned `summary: uploaded=0 skipped=50 failed=0` in dry-run (50 traces processed end-to-end). Default Tempo URL fixed (tempo-query-frontend → tempo, single-binary mode). Helm value pattern adopted (TEMPO_BASE_URL via .Values.tempo.baseUrl). Phase 1 hardening: native GCS SDK to replace gsutil shellout; CronJob template (synth currently invoked via `kubectl run` debug Pod — `make debug-synth-now` produces real GCS uploads). | mikelear/tempo-to-har#1 |
| 2.4-pre / Platform Foundations | ✅ | **complete 2026-05-05** — Tempo + CNPG installed on **both** GCP + AZ clusters (parity, not GCP-only as originally scoped — user-flagged that asymmetric installs hide cross-cloud bugs). Tempo 1.24.4 single-binary, OTLP 4317/4318, 30d retention. CNPG operator 0.28.0 + `leartech-staging` Cluster (1 primary + 1 replica) per cluster. Helm value pattern adopted: `tracing.{enabled,endpoint}` in chart values + per-cluster overlay in `helmfiles/jx-staging/configs/`; `postgresql.useSharedCluster` toggle in auth-service chart (Database CRs + DSN secretRef from CNPG-generated secret). New code-standards rule: `[HIGH] service URLs MUST be env-injected, not hardcoded fallbacks only` (go.md + helm.md). Golden Go template backports OTLP wiring + DB convention docs so every new clone gets it free. 9 PRs merged across 6 repos. | jx-build-cluster-gsm#241,#240; jx-build-cluster-akv#126,#127; leartech-qa-canary#4; tempo-to-har#1; leartech-go-service-template#10; leartech-llm-training-data#4; leartech-auth-service#26 |
| 2.5 | ⏳ | not started | — |
| 2.6 | ⏳ | not started | — |
| 2.7 | ⏳ | not started | — |

### Phase 2.5

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 2.5.1 | ⏳ | not started | — |
| 2.5.2 | ⏳ | not started | — |
| 2.5.3 | ⏳ | not started | — |
| 2.5.4 | ⏳ | not started | — |
| 2.5.5 | ⏳ | not started | — |
| 2.5.6 | ⏳ | not started | — |
| 2.5.7 | ⏳ | not started | — |

### Phase 2.7

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 2.7.1 | ⏳ | not started | — |
| 2.7.2 | ⏳ | not started | — |
| 2.7.3 | ⏳ | not started | — |
| 2.7.4 | ⏳ | not started | — |
| 2.7.5 | ⏳ | not started | — |
| 2.7.6 | ⏳ | not started | — |

### Active sessions (live multi-session view)

When parallel sessions are running, list them here so each session can see ground state at a glance.

```
(none active)
```

Format when active:

```
- 1.3 gate-skeleton — in progress on `mikelear/leartech-gate@feat/skeleton`; first quill green
- 1.2 result-store — done; `pipeline-catalog` v0.5.0 cut; consumers can pin
- 1.6 repo-type-seed — 12/30 repos PR'd; CODEOWNERS approving
```

### Lessons / leverage register

Patterns that landed during sessions, with the marginal cost of next instance. Move stable entries to `~/leartech/hub/shared-rules/platform-leverage.md` once Phase 1 ships.

| Pattern landed | Reusable for | Marginal cost |
|---|---|---|
| _(none yet)_ | | |

---

## How to use this document

**As a planner:**
- Sessions table tells you what's next
- Parallelization view tells you which streams can run concurrently
- Calendar tells you realistic elapsed time

**As a session:**
- Check live-status before starting — know what's in flight
- Update live-status after committing — let other sessions see ground state
- Refer to relevant doc(s) named in the "Read first" section of the brief
- Keep within scope; defer surplus to a later session

**As the human router:**
- 1-2 parallel sessions is sustainable
- 3 is intense but workable
- 4+ becomes triage burden — slow down
- Daily 15-min review of live-status keeps streams aligned
