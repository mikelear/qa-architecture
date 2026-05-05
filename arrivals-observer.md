# `leartech-arrivals-observer` — post-deploy regression detection + traffic forensics

A long-running Go service that watches K8s ReplicaSet `Added` events in `jx-staging`, runs the same Playwright suite that ran pre-merge against the freshly-deployed staging URLs, diffs results against pre-merge baselines, and on regression auto-triggers **traffic forensics** (Tempo span diff between deployment windows) for diagnostic context. Slack-alerts the suspected PR author with both the failed test list and the network-behavior diff.

This is leartech's `mqube-fat-controller` equivalent. **Lifted directly** from mqube's pattern (`~/mqubeRepos/mqube-fat-controller/`) with structural improvements: single result-store (no parallel Mongo), pull-based config from qa-management, integrated traffic forensics that mqube doesn't have.

## Why this exists (the case for post-deploy observation)

PR-time gating (shift-left + risk-assessor + leartech-gate) catches a lot, but cannot catch:

| What slips through PR-time | Why |
|---|---|
| Environmental config drift | Preview env ≠ staging; secrets, env vars, ingress can differ |
| Scale-only failures | Preview is minimal pods; staging has real traffic concurrency |
| Cross-service regressions | Service A change passes A's tests; breaks B's flow at runtime |
| Data-shape issues | Preview uses synthetic data; staging has real-shape data |
| Timing / race conditions | Only manifest under real concurrent load |
| Dependency upgrades passing unit tests but breaking runtime | Even with Renovate hardening |
| Cluster infra drift | A pod that comes up clean in preview can OOMKill in staging |
| Third-party API integration drift | Sandboxes vs real; rate limits; auth flow changes |

This is precisely why mqube's Fat Controller exists, and why removing it would lose the entire post-deploy regression signal class.

## Critical correction to the prior framing

In an earlier version of this architecture I described Fat Controller as "informational, not on the gate's critical path." **That was wrong.** Re-reading my own mqube analysis carefully:

- Porcupine's testing quill (`~/mqubeRepos/porcupine/pkg/quills/testing/usecase.go:91-104`) reads `automated-qa-service.GetTestResults()` filtered to the **staging deployment interval** of the version being promoted.
- Those results were **produced by automated-qa-service**.
- **Fat Controller is what triggers automated-qa-service to run them post-deploy** via its ReplicaSet watcher.

So the chain is:

```
new version deploys to staging
    ↓
Fat Controller's ReplicaSet watcher fires
    ↓
Fat Controller dispatches Playwright tests via K8s Jobs
    ↓
automated-qa-service runs tests, stores results in Mongo
    ↓
later: promotion PR opens
    ↓
porcupine quill reads automated-qa-service results for the staging deployment window
    ↓
no results in window = quill fails = gate blocks
```

**Without Fat Controller, no post-deploy results exist for porcupine to read, and the gate can't validate the staging run.** Fat Controller IS load-bearing for the gate — it's structurally upstream in the data flow, even though there's no direct API call between them.

For our leartech design, the same logic applies: **without `leartech-arrivals-observer`, our gate's `post-deploy-tests` quill has no data to consult**. It's not optional once we add the post-deploy quill.

## Architecture

```
K8s ReplicaSet "Added" event in jx-staging
    │ (client-go informer + Redis distributed lock for dedup)
    ▼
arrivals-observer: create arrival doc in result-store
  gs://test-artifacts-product-first/arrivals/v1/<repo>/<sha>/<env>.json
    │
    ▼
Trigger same Playwright suite that ran pre-merge —
but pointed at STAGING URLs (not preview)
    │  (reuses leartech-pipeline-catalog/end2end-ui task;
    │   parameterized by env var STAGING_URL vs PREVIEW_URL)
    ▼
Results land at:
  gs://test-artifacts-product-first/results/v1/post-deploy/<repo>/<sha>/<cluster>/<test-pack>/<test>.json
    │
    ▼
Diff against pre-merge results for same SHA in result-store:
    pre-merge passed + post-deploy failed → REGRESSION
    pre-merge already failed → not a new regression (suppress)
    pre-merge missing → first-deploy of this SHA, treat as baseline (no diff)
    │
    ├──► No regression → update arrival doc status=Passed
    │
    └──► Regression → auto-trigger traffic-forensics
                ↓
            Query Tempo for spans in v_new deployment window:
              - service-pair edges
              - call rates per edge
              - error rates per edge
              - latency distributions
            Query Tempo for same in v_old (last good) window
            Compute diff (new edges, removed edges, rate shifts, error spikes)
                ↓
            Slack alert in #releases:
              "🚨 Regression detected: leartech-broker-ui v11.1.0
               Failed tests: legal-offering, financial-crime
               Traffic forensics:
                 - NEW edge: broker-ui → notification-service (12 req/min)
                 - notification-service error rate: 0.1% → 8%
               Suspected PR: #4521. Author: @Tony"
                ↓
            Append regression-log entry in qa-management
                ↓
            Update arrival doc status=Failed + forensics_attached=true
                ↓
            leartech-gate at staging→prod hop reads arrival doc:
              post-deploy-tests quill fails → blocks promotion
              (alert-only initially; blocking after baseline period)
```

## Project structure (Go, lifted from mqube-fat-controller)

```
leartech-arrivals-observer/
  cmd/api/main.go                  # entry point — server.Run()
  cmd/cli/                         # ops CLI (manual trigger, status query)
  internal/
    server/
      inject.go                    # DI; matches mqube-fat-controller pattern
      datasources.go               # Redis, GCS, optional Mongo (for arrivals)
    arrivals/
      usecase.go                   # arrival lifecycle; lifted patterns
      handler.go                   # /api/Arrivals API (StructuredSearch, TriggerArrival)
      mongo_repository.go          # if Mongo chosen for arrivals; else GCS
    replicasets/
      usecase.go                   # K8s informer; LIFTED FROM mqube-fat-controller/internal/replicasets/
    testtrigger/
      usecase.go                   # creates K8s Jobs to run end2end-ui task
    testpolling/
      usecase.go                   # polls test results; LIFTED FROM mqube-fat-controller/internal/testpolling/
    forensics/
      usecase.go                   # NEW — Tempo span diff
      tempo_client.go              # uses leartech-go-common Tempo client
    slack/
      usecase.go                   # Slack alerter; LIFTED FROM mqube-fat-controller/internal/slacknotification/
      formatter.go                 # message templates
    config/
      config.go                    # YAML + env vars; pulls qa-management
    domain/
      arrivals.go, replicasets.go, forensics.go, slack.go  # interfaces
  pkg/dtos/
    arrivals.go, forensics.go      # DTOs
  charts/leartech-arrivals-observer/  # Helm chart
  Dockerfile, Makefile, go.mod
```

Approximate code-reuse from mqube-fat-controller: **~60-70%**. The rest is leartech-specific glue (forensics integration, GCS instead of Mongo for results, qa-management config pull).

## Configuration (annotated)

```yaml
# Lifted from mqube-fat-controller-config-cm.yaml with modifications

architect:
  # Architect-equivalent: leartech doesn't have one yet, so use direct GitHub API
  # for PR-author lookup; Tempo for replica-set state
  baseURL: ""

arrivalProcessing:
  enabled: true
  processingTimeoutMinutes: 20    # same as mqube; tunable

github:
  repositoryOwner: spring-financial-group   # leartech's primary org
  organizations:
    - spring-financial-group
    - mikelear

resultStore:
  bucket: test-artifacts-product-first
  arrivalPrefix: arrivals/v1/
  resultsPrefix: results/v1/
  forensicsPrefix: forensics/v1/

testing:
  jobIntervalSeconds: 40           # outer poll loop
  qaTestIntervalSeconds: 15        # inter-job dispatch spacing
  maxRetriesAutomatedQa: 1         # per-test retry; not per-suite
  maxRetriesCronJob: 0

forensics:
  enabled: true
  tempo:
    baseURL: http://tempo.jx-observability:3200
    queryWindowMinutes: 30         # how far back from arrival to query
  diff:
    rateChangeThresholdPct: 25     # call rate must change >25% to flag
    errorRateThresholdPct: 5       # error rate must increase >5% to flag
    minSamples: 10                 # need ≥10 spans per edge to compare

slack:
  enabled: true
  channel: C0LEARTECH_RELEASES     # placeholder
  notificationServiceURL: http://leartech-notification-service:5000   # if/when one exists
  userMappings:                    # GitHub-handle → Slack-user-ID
    mikelear: U...                 # to seed
    # ... populate from actual leartech team

regression:
  newlyFailedOnly: true            # match mqube's behavior: alert on regressions only
  diffWindow:
    lookbackArrivals: 1            # compare to most-recent prior good arrival
```

## Key behaviors (matching mqube + improvements)

### Newly-failed diff (lifted from mqube)

`mqube-fat-controller/internal/arrivals/usecase.go:505-543` — `analyzeFailures()`:

```
For each failed test in current arrival:
  - Look up most recent prior result for same test name across past arrivals
  - If was passing → "newly failed" → alert
  - If was already failing → suppress (no new info)
  - If never run before → "newly failed" (treat as new)
```

We lift this directly. Already-broken tests don't re-alert. Avoids alert fatigue.

### Per-test retry (lifted from mqube)

`maxRetriesAutomatedQa: 1`. A failing test gets retried once in a fresh K8s Job. Implication: tests must be independent (no shared mutable state). We lift this constraint along with the pattern.

### Author missing from userMappings — fixed (vs mqube)

Mqube silently skips Slack notification if author isn't in `userMappings` (their `arrivals/usecase.go:468-471`). **We change this:** if mapping is missing, alert goes to a fallback channel with `@here` instead of @-mention. **No silent misses.**

### Traffic forensics — new (mqube doesn't have this)

On regression, query Tempo for spans in the v_new deployment window vs v_old window. Compute diff:

- **New edges**: service pairs called in v_new but not in v_old
- **Removed edges**: pairs in v_old but not in v_new (often == edges that broke)
- **Rate shifts**: edges with >threshold% change in calls/min
- **Error rate shifts**: edges with >threshold% change in 5xx rate
- **Latency shifts**: edges with p95 change >25%

Output rendered into Slack alert + stored as artifact:

```
gs://test-artifacts-product-first/forensics/v1/<repo>/<sha>/<env>/diff.json
```

Architect-ui (or a future leartech equivalent dashboard) can render this for human triage.

### Production policy — staging-only initially

`leartech-arrivals-observer` watches `jx-staging` only at first. Production observation is DEFERRED:

- Destructive tests must be excluded — only read-paths, idempotent operations
- Test data quarantine — clean up if creating users/cases
- PII exposure — test runs see real customer data
- Rate limiting — high-frequency tests can DoS our own services
- Auth scoping — needs prod credentials with tight scopes

After ~6 weeks of operating on staging, evaluate adding production observation as a separate decision.

## Result schemas

### Arrival doc

```json
{
  "schema_version": "v1",
  "arrival_id": "uuid",
  "service": "leartech-broker-ui",
  "version": "v11.1.0",
  "sha": "abc123def",
  "env": "jx-staging",
  "cluster": "gcp",
  "replica_set_uid": "k8s-uid-here",
  "created_at": "2026-05-04T14:23:01Z",
  "status": "Processing",
  "test_results": [
    {"name": "legal-offering", "status": "Pending", "retry_count": 0, "max_retries": 1}
  ],
  "alert_triggered": false,
  "newly_failed_tests": [],
  "forensics_attached": false,
  "suspected_pr": null,
  "suspected_author": null
}
```

### Forensics diff

```json
{
  "schema_version": "v1",
  "arrival_id": "uuid",
  "service": "leartech-broker-ui",
  "v_new": "v11.1.0",
  "v_old": "v11.0.4",
  "window_minutes": 30,
  "diff": {
    "new_edges": [
      {"from": "leartech-broker-ui", "to": "leartech-notification-service",
       "calls_per_min_new": 12.4, "calls_per_min_old": 0,
       "error_rate_new": 0.08, "error_rate_old": 0,
       "p95_ms_new": 480, "p95_ms_old": null}
    ],
    "removed_edges": [],
    "rate_shifts": [],
    "error_spikes": [
      {"from": "leartech-broker-ui", "to": "leartech-notification-service",
       "error_rate_new": 0.08, "error_rate_old": 0.001, "delta_pct": 7900}
    ],
    "latency_shifts": []
  },
  "summary": "1 new edge, 1 error spike on the new edge. Strong indicator of unpredicted dependency.",
  "rendered_at": "2026-05-04T14:31:22Z"
}
```

## API surface (HTTP, port 8080)

Lifted from mqube-fat-controller's API + extended:

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | liveness/readiness |
| GET | `/metrics` | Prometheus |
| GET | `/swagger` | API docs |
| GET | `/api/Arrivals/StructuredSearch` | paged arrival list (admin) |
| POST | `/api/Arrivals/TriggerArrival` | manually trigger arrival processing (ops escape hatch) |
| GET | `/api/Arrivals/{id}` | fetch single arrival + status |
| GET | `/api/Forensics/{arrival_id}` | fetch forensics diff for an arrival |
| POST | `/api/Forensics/Run` | manually run forensics for a (service, v_new, v_old) tuple |

All authenticated via OAuth2 (matches leartech-go-service-template).

## Integration with `leartech-gate`

New quill in qa-management's `gate-metadata/quills.yaml`:

```yaml
quills:
  post-deploy-tests:
    blocking: false                # alert-only initially; promote to blocking after baseline
    description: "Required Playwright tests passed against the staging deployment of the promoted version"
    impl: result-store-lookup
    config:
      result_store: gs://test-artifacts-product-first/results/v1/post-deploy/
      tolerate_missing_window: true   # if arrivals-observer hasn't observed yet, don't fail
```

Quill behavior:

```
For each promoted service v_new:
  Query result-store for results/v1/post-deploy/<service>/<sha>/...
  If no results found:
    if tolerate_missing_window: pass (with warning in PR comment)
    else: fail
  If results found:
    All required tests passed in window? → pass
    Otherwise → fail (newly-failed list in PR comment)
```

After the first 6-8 weeks of operation, flip `blocking: true` for services where the signal is reliable. Quill metadata supports per-service overrides.

## Triggers

### Primary: K8s ReplicaSet watcher

Long-running Go service, `client-go` informer pattern. Lifted directly from mqube-fat-controller.

### Secondary: manual trigger

`POST /api/Arrivals/TriggerArrival` for ops escape hatch (re-run for a specific replica-set when transient failure is suspected, etc.).

### Tertiary: scheduled (CronJob, optional)

For services where ReplicaSet events aren't reliable (rare, but happens on certain workload types), a scheduled CronJob calls `TriggerArrival` for each known service. Belt-and-braces.

## Notifications via the `notify` framework (decoupled from Slack)

Earlier drafts proposed a "direct Slack webhook" alerter. That's been replaced by the **`leartech-go-common/notify` framework** — see `notifications.md` for the full design. Brief recap:

- `notify.Notifier` interface — one Event type, many transports
- Built-in impls: `SlackNotifier`, `WebhookNotifier`, `LessonCaptureNotifier`, `NoopNotifier`, `FanoutNotifier`
- Routing rules (which transports fire for which event types) live in `leartech-qa-management/notification-config.yaml`
- Per-user identities are platform-agnostic (`{slack: U..., teams: 28:..., email: ...}`)
- "What if leartech doesn't have Slack at all" → `NoopNotifier` or `WebhookNotifier` to whatever the team uses; arrivals-observer doesn't change

Arrivals-observer becomes the framework's **first major consumer**. On regression detection, it builds an `arrivals.regression` Event and calls `router.Notify(ctx, event)`. The router consults `notification-config.yaml` to decide which transports fire. This consumer is now transport-agnostic — adding Teams, removing Slack, or fanning to multiple destinations is a config change, not a code change.

### Event types arrivals-observer emits

| Event type | When | Default routing |
|---|---|---|
| `arrivals.regression` | Newly-failing test detected vs pre-merge baseline | `[slack]` always; `[lesson-capture]` if author is automated-agent |
| `arrivals.timeout` | 20-minute processing timeout hit | `[slack]` |
| `arrivals.processed` | Successful arrival completion | (silent by default; opt-in via routing config if dashboards want it) |

### Why the framework over a direct webhook

Three reasons made the framework the right call before Phase 2.7 ships:

1. **`leartech-gate`, `risk-assessor`, AI review, Renovate-hardening** all want notifications too. Building the framework once and reusing it is cheaper than each consumer wiring its own Slack webhook.
2. **Pluggability** — leartech might move off Slack to Teams, or want PagerDuty for high-severity. With the framework, that's a routing change in qa-management; with direct webhooks, it's a multi-repo refactor.
3. **Lesson-capture transport** — needed for the `~/leartech/automated-agent/` integration (see below). Cleanly slots in as another transport rather than a bespoke arrivals-observer code path.

## Integration with `~/leartech/automated-agent/` lesson capture

When `automated-agent` (the auto-PR system) opens a PR and that PR causes a post-deploy regression, the regression should flow back into the agent's lesson-calibration system so it learns to avoid the class of mistake on future runs. This is the **L4 feedback loop** the agent's lessons schema was designed for.

But: not every regression is agent-relevant. Some are caused by human PRs. Some are flakes. Some are env config drift unrelated to the change. So feeding ALL regressions into the lesson system would be noise. Two safeguards:

### Safeguard 1: Filter on PR author

The `arrivals.regression` routing rule (in `notification-config.yaml`) fires `[slack]` always but only fires `[lesson-capture]` when `author.is_automated_agent: true`:

```yaml
arrivals.regression:
  transports: [slack]
  conditional_transports:
    - when:
        author.is_automated_agent: true     # filter on the configured agent identity
      transports: [lesson-capture]
```

The `is_automated_agent` predicate matches against the `automated_agents.github_handles` list in qa-management — a small allowlist of bot identities (e.g. `automated-agent-bot`, `leartech-bot`).

A regression caused by a human's PR fires Slack but never reaches the lesson system. A regression caused by `automated-agent-bot`'s PR fires both.

### Safeguard 2: `status: candidate` not `status: open`

Even within agent-PRs, not every regression is a real lesson. Some are flakes, env drift, or pre-existing bugs the agent's PR merely exposed. So `LessonCaptureNotifier` writes lessons with **`status: candidate`**, not `status: open`. A human (or designated reviewer bot) periodically triages candidates:

- Promote to `status: open` → enters active calibration queue; agent learns from it
- Discard → not a calibration signal; logged as "auto-captured, judged not relevant"

The triage burden is small (probably <30 min/week — most candidates are obvious agent-mistakes or obvious flakes). Avoiding it would mean agent calibration absorbs noise.

**Important**: this requires `automated-agent`'s lesson schema to support `status: candidate` (or equivalent). If it currently only supports `status: open`, a small schema extension on the agent side is needed before Phase 2.7's `LessonCaptureNotifier` ships. Tracked as Q-AO7 in `open-questions.md`.

### Manual capture path stays available

For QA-analysis deep-dive sessions that want curated lesson capture (rather than the auto-path):

```bash
cd ~/leartech/automated-agent
uv run lessons capture \
  --title "Modal flicker on Safari only — slipped past Playwright + manual review" \
  --source-type prod_incident \
  --source-reference INC-1234 \
  --source-observer qa-analysis-session \
  --category criteria_gap \
  --status open                    # human is curating; skip the candidate step
```

Source-type mapping:

| Observation | `--source-type` | Typical category |
|---|---|---|
| Found while testing on staging before prod promotion | `staging_test` | `criteria_gap` |
| Bug reported post-prod via incident tracker | `prod_incident` | `criteria_gap` |
| Manual reviewer flagged in PR comments before merge | `manual_review` | `criteria_gap` |
| Pattern observed across multiple incidents (root cause) | `prod_incident` | `architecture` |

Each lands as a frontmatter+md file in `~/leartech/automated-agent/gate/agent/lessons/catalog/`. Same schema regardless of source.

The auto-path is for **breadth** (catch every agent-PR regression so nothing slips). The manual path is for **depth** (curated findings from a specific deep-dive). Both coexist; they serve different needs.

## Build estimate (with leverage)

| Piece | Effort | Reuses |
|---|---|---|
| Repo skeleton from leartech-go-service-template | half day | golden Go template |
| K8s ReplicaSet watcher | 2-3 days | direct lift from mqube-fat-controller/internal/replicasets/ |
| Redis distributed lock for dedup | half day | mqube pattern, leartech-go-common candidate |
| Arrival doc schema + GCS write | 1 day | result-store extension |
| Test trigger via K8s Job → end2end-ui task | 2 days | parameterize PREVIEW_URL → STAGING_URL on existing task |
| Result polling + newly-failed diff | 1 day | direct lift from mqube-fat-controller |
| Tempo span query client | 1 day | leartech-go-common candidate |
| Forensics diff engine (edges, rates, errors) | 2-3 days | NEW; rule-based; tunable thresholds in config |
| Slack alerter (option 2 — direct webhook) | 1 day | mqube formatter is clean reference |
| Slack message rendering with forensics diff | 1 day | NEW |
| `post-deploy-tests` quill in leartech-gate | half day | extends data-driven quill framework |
| Helm chart, Tekton wiring, deploy to jx-staging | 1 day | golden patterns |
| Docs, runbook, dashboards | 1 day | — |

Total: **~15 days = ~3 weeks** for one engineer; ~2 weeks parallelized across 2.

## Sessions plan (Phase 2.7)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.7.1** | Repo skeleton + K8s watcher | Go service from template; ReplicaSet informer working against jx-staging in dev cluster; Redis lock; arrival doc written to GCS on event | new repo: `mikelear/leartech-arrivals-observer` | 4-5h |
| **2.7.2** | Test trigger + result polling | K8s Job dispatch via end2end-ui task pointed at staging; result polling loop; newly-failed diff vs pre-merge | `leartech-arrivals-observer` + small extension to `leartech-pipeline-catalog/tasks/end2end-ui/` for STAGING_URL parameter | 4-5h |
| **2.7.3** | Tempo client + forensics engine | Tempo span query; edge graph extraction; diff engine with rule-based thresholds; forensics doc written to GCS | `leartech-arrivals-observer` + `leartech-go-common/tempo` | 5-6h |
| **2.7.4** | Slack alerter + integration | Direct Slack webhook (option 2); message template with forensics rendering; userMappings config; fallback to @here | `leartech-arrivals-observer` | 3-4h |
| **2.7.5** | post-deploy-tests quill | Quill in leartech-gate; alert-only initially; integration tests | `leartech-gate` | 2-3h |
| **2.7.6** | Phase 2.7 retro | Validation criteria; calibration of forensics thresholds; risk-assessor calibration loop (see below) | hub | 2h |

## Calibration feedback loop with risk-assessor

A subtle benefit: traffic forensics surfaces **edges that risk-assessor's static analysis didn't predict**. Pattern:

```
Risk-assessor predicted: PR affects A, B, C
Arrivals-observer detects: regression in B's tests
Traffic-forensics shows: regression edge is actually broker-ui → notification-service
                          (notification-service was NOT in risk-assessor's prediction set)
```

This means **risk-assessor's static analysis missed an edge**. Track this as a metric:

```
unpredicted_edge_rate = forensics-revealed-edges-not-in-static-prediction / total-regressions
```

If consistently >20%, risk-assessor's static rules need improvement. Common patterns:
- Shared config repo's effects under-modeled (a config change reaches more services than the AST shows)
- Dynamic dispatch / reflection / message-bus consumers (Go interfaces, dependency injection, kafka topics)
- Generated code consumers (the AST sees the generation source but not the generated callers)

Each pattern, once identified, becomes a new rule in `risk-assessor`'s service-catalog or risk-config. **The system gets smarter via operational feedback** rather than via hand-tuning.

In the Phase 2.7 retro, track `unpredicted_edge_rate` and treat as a Phase 2.5.7 calibration input.

## Open questions

See `open-questions.md`. Arrivals-observer-specific:

- **Q-AO1**: Mongo or GCS for arrival docs? (Lean GCS for consistency; Mongo if dashboards need richer queries — same trade-off as result-store.)
- **Q-AO2**: How long to retain forensics artifacts? (Lean 90 days like result-store; longer for "interesting" regressions.)
- **Q-AO3**: When to flip `post-deploy-tests` quill from alert-only to blocking? (Suggest after 6-8 weeks of stable operation; per-service decision.)
- **Q-AO4**: Production observation policy — when, what tests are safe? (Defer to a separate Phase 3 design pass.)
- **Q-AO5**: Should arrivals-observer also emit the regression-log in qa-management as a structured PR? (Propose: yes, but auto-merge for `add-only`, like coverage scanner.)

## What this gives us that mqube doesn't have

- **Traffic forensics on regression** — diagnoses WHY a regression happened, not just THAT it happened
- **Single result-store** — pre-merge and post-deploy results in one place, gate reads both
- **No silent missing-userMapping skip** — fallback to channel @here
- **Calibration feedback loop with risk-assessor** — the system gets better at prediction over time
- **Pull-based config from qa-management** — observer's behavior tunable without rebuild
