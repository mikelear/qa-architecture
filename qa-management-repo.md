# `leartech-qa-management` — the canonical QA config repo

The single source of truth for everything QA-related: required tests per service, repo-type policy references, load-test SLAs, gate metadata, override audit log. All other consumers (gate, test runners, load tester, AI coverage scanner) read from this repo via Renovate-pinned tags.

This is the structural fix for mqube's biggest operational pain — their three-place test-list drift (`mqube-fat-controller/data/testingConfiguration.yaml` + `mqube-/mpowered-automated-qa-config` + `JX3_Azure_Vault_Production/.porcupine/quills/testing.yaml`). One repo, many consumers, pull semantics.

## Repo layout

```
leartech-qa-management/
  README.md
  CLAUDE.md
  
  required-tests/
    <service-name>.yaml         # per-service required test list (gate input)
  
  repo-type-policy/
    test-packs.yaml             # which test packs each repo type must run + risk_modifiers
    deferral-rules.yaml         # `coverage-deferred` policy (TTL, approver list)
  
  service-catalog.yaml          # service ownership + dependency graph (risk-assessor input)
  risk-config.yaml              # risk-classifier thresholds and factor weights (risk-assessor input)
  notification-config.yaml      # transports, users (multi-platform identities), routing rules — Notifier framework input
  
  load/
    <service-name>.yaml         # per-service load-test SLA configuration
  
  gate-metadata/
    quills.yaml                 # which quills run, blocking vs alert
    overrides.log.jsonl         # audit log of /override usage (CI-appended)
  
  test-catalog/
    <test-name>.yaml            # canonical test metadata: owner, repo, type, tags
  
  .lighthouse/jenkins-x/        # this repo's own CI (lint, schema validate)
    pullrequest.yaml
    triggers.yaml
  
  schemas/
    *.cue                       # CUE schemas for every YAML file type (CI validates)
  
  Makefile                      # `make validate` runs all CUE schemas
  charts/                       # if we publish as a Helm chart for consumers (TBD)
```

## Schemas (concrete, paste-ready)

### `required-tests/<service>.yaml`

```yaml
service: leartech-broker-ui
type: ui-mfe                          # cross-ref to repo-type-policy
required_tests:
  - name: broker-registration-da
    blocking: true                    # gate fails if this missing
    quarantined: false                # exclude from gate without removing
    flake_budget: 0.05                # auto-quarantine if pass-rate < 95%
    owner: platform-broker
    suite: broker-onboarding
  
  - name: legal-offering
    blocking: true
    quarantined: false
    flake_budget: 0.05
    owner: platform-broker
    suite: full-auto
  
  - name: broker-edge-condition-x
    blocking: false                   # alert-only, doesn't block
    quarantined: true                 # currently flaky; tracked for fix
    quarantined_until: 2026-06-01     # auto-unquarantine after this date
    quarantined_reason: "BR-1247 — env-specific timing issue"
    owner: platform-broker
    suite: edge-cases
```

**Solves**: chronic flakes blocking every promotion (mqube's pain). Quarantine + flake_budget = explicit safety valve, auditable.

### `repo-type-policy/test-packs.yaml`

```yaml
# Repo types and their required test packs.
# .leartech/repo-type.yaml in each service repo declares its type;
# AI code reviewer + Tekton triggers consult this file to know what to run.
#
# risk_modifiers extend the base required_packs based on the leartech-risk-assessor
# output for the PR. See risk-assessor.md for the assessment schema.

repo_types:
  ui-mfe:
    required_packs:                       # base (low risk gets just these)
      - playwright-ui
      - accessibility                     # axe-core in same Playwright runs
    optional_packs:
      - lighthouse-perf
      - visual-regression
    risk_modifiers:
      low:
        # base only
      medium:
        add: [contract, bundle-size-budget]
      high:
        add: [contract, bundle-size-budget, lighthouse-perf]
        require_human_review: true
        require_two_key_override: true
  
  go-service:
    required_packs:
      - unit
      - contract                          # OpenAPI conformance
    optional_packs:
      - k6-smoke
      - dast
    risk_modifiers:
      low:
        # base only
      medium:
        add: [integration]
      high:
        add: [integration, dast]
        require_human_review: true
        require_two_key_override: true
  
  config-rules:
    required_packs:
      - schema-validate
      - back-testing
    optional_packs:
      - shepherd-coverage
    risk_modifiers:
      low:
        # base only
      medium:
        add: [shepherd-coverage]
      high:
        add: [shepherd-coverage]
        require_human_review: true
  
  helm-chart:
    required_packs:
      - helm-lint
      - kube-test
      - render-dry-run
    # No risk modifiers — chart changes already produce deterministic diffs;
    # risk is captured by the changes themselves, not by what's around them.
  
  pipeline-catalog:
    required_packs:
      - preview-shift-left-task-tests
  
  golden-template:
    required_packs:
      - self-test                         # scaffold → build → run scaffolded tests
```

**Solves**: opaque coverage expectations. Engineers can read this file to know "for my repo type, what's required". AI reviewer cites this rule directly in PR feedback.

### `load/<service>.yaml`

```yaml
service: leartech-broker-ui
har_source: playwright              # one of: playwright, tempo, hubble, manual
har_filter:
  exclude_suffixes: [.js, .css, .png, .jpg, .jpeg, .gif, .woff2, .svg]
  include_hostnames: ["*.jx.leartech.build", "*.leartech.com"]

sla:
  p95_latency_ms: 500
  p99_latency_ms: 1500
  error_rate_max: 0.01
  vs_baseline:
    metric: p95_latency_ms
    max_regression_pct: 10          # fail gate if latency increases >10% vs prior release

shape:
  ramp:
    - {duration: 5m, target_rps: 50}
    - {duration: 20m, target_rps: 200}
    - {duration: 5m, target_rps: 0}

trigger:
  cron: "0 2 * * *"                 # nightly 02:00 UTC
  on_promotion: true                # also runs as part of leartech-gate
```

**Solves**: load-test signal that's actually gateable. mqube's load-testing-results store has no SLA assertions today.

### `notification-config.yaml`

Drives the `leartech-go-common/notify` framework — see `notifications.md` for full design. Transports, per-user multi-platform identities, and event-routing rules. Pull-based; consumers (arrivals-observer, gate, risk-assessor) Renovate-pin to a tag.

```yaml
# leartech-qa-management/notification-config.yaml
schema_version: v1

transports:
  slack:
    webhook_secret_ref: slack-webhook       # K8s secret name in consumer's namespace
    default_channel: "C0LEARTECH_RELEASES"
  
  lesson-capture:
    agent_root: /var/run/leartech-automated-agent
    default_status: candidate                # NOT `open` — humans triage to active queue
    default_observer: arrivals-observer
  
  noop: {}                                  # always available; logs only

users:
  mikelear:
    github: mikelear
    slack: U02ABCDEF
    teams: 28:abc-..-def
    email: mike.lear@leartech.com
  # ... per-user multi-platform identity dict

automated_agents:
  github_handles:                           # PR authors that are bots, not humans
    - automated-agent-bot
    - leartech-bot

routing:
  arrivals.regression:
    transports: [slack]                     # always — humans see all regressions
    fallback_tag: "@here"
    
    conditional_transports:
      - when:
          author.is_automated_agent: true   # only fire lesson-capture for agent-PRs
        transports: [lesson-capture]
        config:
          source_type: ${event.tags.env_classification}   # "staging_test" or "prod_incident"
  
  arrivals.timeout:
    transports: [slack]
  
  gate.override:
    transports: [slack]
    only_when: {severity: error}
  
  risk.high:
    transports: []                          # silent for now; flip to [slack] if signal becomes useful
  
  ai-review.score-low:
    transports: [slack]
    only_when: {score: "<60"}
  
  renovate.full-qa-fail:
    transports: [slack]
```

**Solves**: Slack lock-in, multiple consumers reinventing webhook handling, automated-agent integration noise. See `notifications.md` for the full framework.

### `gate-metadata/quills.yaml`

```yaml
quills:
  shift-left-tests:
    blocking: true
    description: "Required Playwright + contract tests passed for the promoted SHA"
    impl: result-store-lookup
    config:
      result_store: gs://test-artifacts-product-first/results/v1/
  
  contract-tests:
    blocking: true
    description: "OpenAPI conformance for promoted services"
    impl: result-store-lookup
  
  co-promotion:
    blocking: true
    description: "Co-required service pairs promote together"
    impl: helmfile-diff
    config:
      pairs:
        - [leartech-case-service, leartech-orchestrator]
  
  migrations:
    blocking: true
    description: "Forward + rollback migrations passed"
    impl: result-store-lookup
  
  load-sla:
    blocking: false                 # alert-only initially; flip to blocking after baseline collected
    description: "Load-test SLAs met, no regression vs baseline"
    impl: result-store-lookup
    config:
      sla_source: load/<service>.yaml

policy:
  override:
    plugin: lighthouse-override     # /override <quill-name>
    audit_log: gate-metadata/overrides.log.jsonl
    approver_required:
      - blocking: true              # blocking-quill overrides need approver
        from_codeowners: true
```

**Solves**: porcupine's "all quills are hardcoded Go" coupling. Quills are config; new quills can be added without modifying gate code if they fit `result-store-lookup` or `helmfile-diff` impls.

## CI on this repo

`leartech-qa-management` is its own product — needs strong CI:

| Presubmit | What it does |
|---|---|
| `validate` | Runs `make validate` — every YAML file passes CUE schema in `schemas/` |
| `cross-reference` | Every test name in `required-tests/*.yaml` exists in `test-catalog/`. Every service name maps to a real source-config entry. Every repo type referenced exists in `repo-type-policy/test-packs.yaml`. |
| `additions-only-auto-merge` | If PR diff is purely additions (new entries, no modifications/removals) AND author is the AI bot, label `auto-merge-eligible` and Tide auto-merges on green |
| `removals-need-review` | If PR removes/modifies entries, requires CODEOWNERS review |
| `lint` | YAML formatting, CUE comment style |

Postsubmit:

| Step | What |
|---|---|
| `cut-tag` | Conventional-commit-driven semver tag (e.g. `v1.42.0`) |
| `publish-chart` | (TBD) Publish as Helm chart so consumers can pin via Renovate |

## How consumers pull

Pull semantics — every consumer pins to a specific tag and bumps via Renovate. **No fan-out push from the management repo.**

| Consumer | How it consumes |
|---|---|
| `leartech-gate` Tekton task | Reads `required-tests/*.yaml` + `gate-metadata/quills.yaml` at run time. Pinned via Renovate-managed image tag (the gate is built with a specific qa-management commit baked in) OR runtime fetch of pinned tag |
| QA test runner (Playwright service) | Init container fetches pinned tag (mqube pattern: `mqube-config-loader` reading `gitRepositoryURL + tag`) |
| `leartech-load-testing` | Same init-container pattern for `load/*.yaml` SLAs |
| AI coverage scanner | Reads `repo-type-policy/test-packs.yaml` + `test-catalog/` to know what's expected |

Renovate (already self-hosted in leartech) auto-opens PRs to bump consumers when qa-management cuts a new tag. Each consumer's PR runs its own checks before merging — staged rollout, audited.

## PR-time companion-PR pattern

When a service PR introduces new test surface (new endpoints / routes / screens), the AI code reviewer:

1. Detects the surface change (Go: AST walk for new routes; Angular: diff routing config; OpenAPI: diff vs main)
2. Composes a **companion PR** against `leartech-qa-management`:

   ```
   Title: "Coverage proposal for leartech-broker-ui#4521"
   Branch: chore/coverage-add-broker-edge-conditions
   Body (structured):
     ## Source PR
     leartech-broker-ui#4521 (branch: feat/edge-broker-handling)
     
     ## Added (auto-mergeable on source PR merge)
     - test_catalog: new entry for `broker-edge-condition-y`
     - required_tests: register `broker-edge-condition-y` for leartech-broker-ui
     
     ## Registered (existing tests, never registered)
     - none
     
     ## Removed (NEEDS REVIEW)
     - none
   Labels: chore/coverage-add, depends-on:leartech-broker-ui#4521
   ```

3. The companion PR depends on the source PR via the labels system. When the source PR merges, the companion auto-merges (Lighthouse Tide handles the dependency).

4. If the source PR introduces surface but the author hasn't added tests AND hasn't applied `coverage-deferred` label → AI reviewer's gate fails the source PR.

5. `coverage-deferred` requires:
   - Approver from CODEOWNERS
   - Rationale in PR body
   - TTL (default 30 days) — after which the deferral expires and the PR re-blocks
   - Logged to `gate-metadata/deferrals.log.jsonl` for audit

This is the **gate-keeping moment**. Release-time scans are cleanup; weekly cron reconciles drift; PR-time is where coverage actually lands.

## Repo bootstrap (for existing leartech repos)

Existing repos don't have `.leartech/repo-type.yaml`. Three-step rollout:

1. **Golden templates seed it for new services** — golden Go template + Angular template ship with `.leartech/repo-type.yaml` already populated. New services come pre-classified.
2. **AI-suggested for existing repos** — one-time bulk pass: classifier reads existing repo (file types, package.json, Chart.yaml etc.) and proposes a `repo-type.yaml`. Open as PR per repo. CODEOWNERS confirm.
3. **Manual fallback** — if AI gets it wrong, the file is overridable.

Estimate: ~3 days for bootstrap pass across the leartech repo fleet (assume ~30 active service repos; classifier handles bulk).

## Why this structure beats mqube's

| Concern | mqube | This proposal |
|---|---|---|
| Test list locations | 3 repos | 1 repo |
| Drift detection | manual via `check-new-releases` warning comment | schema CI cross-references; impossible to commit broken state |
| Coverage gap caught | at production-promotion-PR (post-merge) | at service PR (pre-merge) |
| Author context when gap caught | days/weeks later | same PR, same author, full context |
| Quill set | hardcoded in porcupine Go | data-driven config in `gate-metadata/quills.yaml` |
| Flake handling | manual override every prod-promotion PR | structured quarantine + auto-quarantine on flake_budget breach |
| Override audit | scattered PR comments | append-only log in `gate-metadata/overrides.log.jsonl` |
| Cross-cluster gate logic | n/a (mqube one cluster gates) | `result_store` query consults both gcp + az tags |

## Open decisions for this component

See `open-questions.md` — relevant ones:

- Helm chart vs raw repo for consumer publishing
- CUE vs JSON Schema for validation (CUE is stricter; JSON Schema has wider tooling)
- How to integrate `gate-metadata/overrides.log.jsonl` updates with Lighthouse override events (webhook? scheduled scan?)
