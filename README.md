# Leartech QA Architecture

Forward-design proposal for leartech's QA + release-gating system. Adapts the mqube pattern (test runner + gate + topology + load testing) but with structural improvements: single source of truth, PR-time gating, pull-based propagation, multi-source HAR pipeline.

## End-to-end flow (the proposal)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Per-repo: .leartech/repo-type.yaml                    │
│                       (golden templates seed it)                            │
│                       Hub shared rule: test-packs-by-repo-type.md           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
              ┌─────────────────────────────────────────────┐
              │    SERVICE PR opens (e.g. mqube-rules-eq)   │
              └─────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┴───────────────────────────┐
        │                                                       │
        ▼                                                       ▼
┌──────────────────────────┐                ┌────────────────────────────────────┐
│ AI code reviewer         │                │ Tekton: required test packs run    │
│ - reads repo-type.yaml   │                │ (per shared rule):                 │
│ - detects new test       │                │   - playwright-ui (if ui-mfe)      │
│   surface (endpoints,    │                │   - contract (openapi conform.)    │
│   routes, screens)       │                │   - unit / integration             │
│ - flags missing coverage │                │   - load-smoke (Phase 2)           │
│ - proposes COMPANION PR  │                │ Results → result-store keyed by SHA │
│   to qa-management repo  │                └────────────────────────────────────┘
└──────────────────────────┘                                  │
        │                                                       │
        ▼                                                       │
┌──────────────────────────┐                                  │
│ leartech-qa-management   │                                  │
│ COMPANION PR (linked):   │                                  │
│ - registers new tests    │                                  │
│ - updates required-tests │                                  │
│ - schema CI validates    │                                  │
│ - auto-merge on source   │                                  │
│   PR merge if `added`    │                                  │
│   only; manual review    │                                  │
│   for `removed`          │                                  │
└──────────────────────────┘                                  │
        │                                                       │
        └───────────────┬──────────────────────────────────────┘
                        │
                        ▼   merge service PR + companion PR
              ┌─────────────────────────────────┐
              │   service release pipeline      │
              │   builds image, opens promotion │
              │   PR to GitOps repo             │
              └─────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────────────────┐
              │ leartech-gate (Tekton presubmit │
              │ on prod GitOps repo)            │
              │                                 │
              │ Quills (data-driven):           │
              │   - shift-left-tests            │
              │     (result-store green for SHA)│
              │   - contract-tests              │
              │   - co-promotion                │
              │   - migrations                  │
              │   - load-sla (Phase 2)          │
              │                                 │
              │ Reads pinned qa-management tag  │
              │ via Renovate. Quill verdicts    │
              │ → fail check on regression.     │
              │                                 │
              │ Override: /override leartech-gate│
              └─────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────────────────┐
              │   merge → bootjob applies →     │
              │   service deploys to prod       │
              └─────────────────────────────────┘

Independent loops feeding qa-management:
  - Weekly cron: full reconciliation (drift catch)
  - Release postsubmit: lighter sweep for renames/deletes
  - Tempo→HAR producer: synthesizes coverage-gap proposals from observed traces
  - Hubble→HAR producer (Phase 3, only if Tempo gaps): same role, broader coverage

HAR pipeline (load testing + future consumers):
  Producers → sanitize/template → replay engine
    Producers:
      - Playwright recordings (default; already free)
      - Tempo span synthesis (Phase 2; reuses existing OTel)
      - Hubble eBPF flows (Phase 3; only if needed)
    Sanitize: strips auth/PII, parameterizes ephemeral IDs
    Replay: leartech-load-testing service (lift mqube pattern)
    SLAs: leartech-qa-management/.load/<service>.yaml; gate quill checks regression
```

## Component summary

| Component | Role | Lift / Build / Reuse |
|---|---|---|
| `leartech-qa-management` (repo) | Single source of truth: required-tests, repo-type policy refs (with risk_modifiers), service catalog, load SLAs, gate metadata | **Build** — new repo, ~1 week |
| `leartech-gate` (Tekton task + Go binary) | Porcupine-equivalent. Reads qa-management, verifies result-store green, fails check | **Lift pattern from mqube porcupine**; ~3 days with golden Go template |
| Result store (extends `gs://test-artifacts-product-first/`) | Per-SHA test verdict storage; consumed by gate | **Extend existing** — bucket + auth pattern already in production via `end2end-ui` task; ~2 hours to add SHA-keyed `results/v1/` prefix |
| QA-coverage-scanner (Tekton task) | PR-time scanner; detects new test surface; opens companion PRs | **Build** — ~1 week (reuses leartech-ai-classifier) |
| `leartech-load-testing` (Go service) | HAR replay engine. SLA-asserted. Triggered by CronJob + Tekton step | **Lift mqube-load-testing chart pattern**; ~3 days + 2 days for SLA quill |
| `tempo-to-har` (Go service) | Phase 2 — synthesizes HAR from Tempo spans for coverage-gap + risk-assessor input | **Build** — ~3 days (uses Tempo's HTTP API + Go templates) |
| `leartech-risk-assessor` (Tekton task + Go service) | Phase 2.5 — classifies PR risk via static (AST) + Tempo + HAR signals; modifies required-tests | **Build** — ~3 weeks; see `risk-assessor.md` |
| Repo-type policy (in qa-management) | `repo-type-policy/test-packs.yaml` — base required packs + risk_modifiers per repo type | **Build** — half-day (YAML schema in qa-management) |
| Renovate full-QA harness | Renovate PRs run full Playwright suite, not just unit tests | **Configure** — ~half day (Renovate + Lighthouse trigger) |
| AI-suggested PR-to-management | Companion-PR opener fed by code-reviewer | **Reuse** — leartech-ai-classifier already running; ~2-3 days for the bridge |

## What this gets you

- **No test-list drift** — one source of truth, consumers pull via Renovate. Solves mqube's biggest operational pain (their N6).
- **PR-time gate-keeping** — coverage caught before merge, with author context. mqube only catches at production-promotion-PR time, by which point fixes are expensive.
- **HAR as multi-consumer asset** — load testing is one consumer; test generation, contract derivation, coverage reporting are future cheap additions.
- **Renovate hardening** — Renovate PRs run the full Playwright suite. mqube doesn't have this.
- **Tempo-first traffic mapping** — uses observability infra you already have. No CNI swap, no new operational stack to learn.

## What this gives up vs. mqube

- **Per-version post-deploy signal** — leartech-gate reads PR-time results, not deployment-time. If a regression only manifests in real cluster conditions (network, secrets, scaled load), shift-left tests miss it. Mitigation: synthetic prod probes (Tekton CronJob hitting prod endpoints; results to result-store).
- **Slack-alert culture** — Fat Controller's main visible output. Not built here; can be added later as a thin notifier reading the result-store.
- **Cronjob health gating** — porcupine's third quill. Worth re-considering only if leartech grows cronjob-heavy services (currently doesn't).

## Foundation leverage

Estimates assume reuse of:

- `leartech-go-service-template` + `leartech-go-common` — every new Go service starts QA-instrumented
- `leartech-pipeline-catalog` — `uses:` pattern for new Tekton tasks; no rebuilds
- `preview-shift-left` — kind+Tekton harness for fast iteration on new tasks
- `leartech-ai-classifier` + `code-standards` LLM infra — drives coverage-suggestion AI without building a new model
- Self-hosted Renovate — already scanning all leartech repos weekly
- `multi-cluster-jx3-pattern` shared rule + hub `pr-pipelines.sh` — multi-cluster gate logic is one helper call
- Grafana stack (Tempo, Loki, Prometheus, Grafana) — already configured per `golden-service-standard.md`

If any of these weren't already in place, the build estimate doubles or triples. Tracking what each new pattern unlocks is worth a hub shared rule (`platform-leverage.md`) the next time we land a foundational capability.

## Phased build

See `build-plan.md` for detail. Headline:

- **Phase 1** (~2 weeks): qa-management repo + leartech-gate + shift-left-tests quill + result-store (extends existing GCS bucket — ~2 hours, not 1 day). Closed loop.
- **Phase 2** (~2 weeks): Tempo-to-HAR + load-testing service + load-sla quill + Renovate hardening + AI coverage suggester.
- **Phase 2.5** (~3 weeks): risk-assessor — static analysis primary + Tempo/HAR confirming; risk-modifier extension to required test packs; gate override-stiffening on high-risk PRs.
- **Phase 3** (later, only if needed): Pixie or Hubble for traffic mapping; synthetic prod probes; Slack-alert layer.

## Open decisions

See `open-questions.md`. Highlights:

- Where does the result-store live (GCS / Mongo / GitHub artifacts)?
- Does the gate need to gate on BOTH clusters (gcp + az) or one?
- Do we want a Slack-alert layer in Phase 1 or defer?
- How do we bootstrap repo-type.yaml across existing repos (manual seed, AI-suggested, golden-template default)?

## Reading order if new to this

1. `~/leartech/Qa-Analysis/README.md` — what mqube does (so you know what we're adapting)
2. This file — the leartech version (what's the same, what's different)
3. `qa-management-repo.md` — the canonical config repo (the load-bearing piece)
4. `gate.md` — the porcupine-equivalent
5. `har-pipeline.md` — load-testing + traffic mapping
6. `risk-assessor.md` — Phase 2.5 risk-based gating (static + Tempo + HAR)
7. `build-plan.md` — sequencing
