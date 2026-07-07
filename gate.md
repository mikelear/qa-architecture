# `leartech-gate` — the porcupine-equivalent

A Go CLI run as a Tekton presubmit on the leartech production GitOps repo. Reads `leartech-qa-management` for required-tests + quills + load SLAs, queries the result store for verdicts, fails the `leartech-gate` PR check if any blocking quill fails. Override via `/override leartech-gate` (Lighthouse `override` plugin).

Direct lift of mqube's porcupine pattern — well-validated; we just simplify and decouple from mqube-specifics.

## What it gates

The promotion PR from staging/preprod → production. Service PRs are gated separately at PR-time via shift-left + AI coverage scanner; this gate is the **second** layer that says "everything tested for this SHA is green; let it ship."

```
Service PR → shift-left tests run → result-store written
Service PR merges → release pipeline → opens promotion PR to GitOps repo
                                       │
                                       ▼
                          Lighthouse fires presubmits:
                          - verify (helmfile + kubetest)
                          - lint-overrides (if configs/ changed)
                          - leartech-gate          ← THIS
                          - beaver-notes (release-notes generator, if adopted)
                          - check-new-releases (release-count drift; if adopted)
                                       │
                                       ▼
                          all pass → Tide merges → bootjob applies → prod
                                       │
                                       └─ any fail → blocked
                                          /override <check-name> bypasses
```

## Internal structure (Go)

```
leartech-gate/
  cmd/api/main.go               # CLI entry
  internal/
    config/                     # loads leartech-qa-management YAML
    helmfile/                   # parses promotion-PR helmfile diff via beaver lib
    quills/
      shiftleft/usecase.go      # data-driven; reads result-store
      contract/usecase.go       # data-driven; reads result-store
      copromotion/usecase.go    # pure helmfile diff check
      migrations/usecase.go     # data-driven; reads result-store
      loadsla/usecase.go        # reads result-store + SLA config
    resultstore/                # GCS/Mongo client interface
    github/                     # PR comment + check status
  pkg/dtos/                     # quill response types
  charts/leartech-gate/         # if we publish for downstream chart users
  Dockerfile
  Makefile
```

Reuses leartech-go-common patterns: structured logging via slog → Loki, OAuth client for any service calls, OpenTelemetry instrumentation, Prometheus metrics endpoint.

## Quill execution model (data-driven)

Mqube's porcupine has each quill as a Go struct with hardcoded config-file path. We improve: quills are **driven by `gate-metadata/quills.yaml`** in qa-management. Three impl backends initially:

```go
type QuillImpl interface {
    Run(ctx context.Context, promotion ServicePromotion, config QuillConfig) QuillResult
}

// Three impls cover all current quills:
type ResultStoreLookup struct { ... }   // shift-left, contract, migrations, load-sla
type HelmfileDiff struct { ... }         // co-promotion
type CronJobHealth struct { ... }        // (Phase 3, only if needed)
```

A new quill = a new entry in `gate-metadata/quills.yaml` pointing at one of these impls + the impl's config. No Go code change for most additions.

## Result store contract

Result store keyed by **commit SHA + cluster tag**. Multi-cluster aware out of the box.

**Bucket already exists**: `gs://test-artifacts-product-first/` is provisioned and in use by `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml` for Playwright artifact uploads (screenshots, videos, traces). Auth pattern is settled — `test-artifacts-gcs-key` Kubernetes secret mounted via ExternalSecret at `/var/run/secrets/test-artifacts/key.json`. We extend that bucket with a `results/` prefix for verdict JSONs:

```
gs://test-artifacts-product-first/
  ├─ <repo>/pr-<N>/<cluster>/<ts>/<testname>--<file>     ← existing (PR-keyed artifacts)
  └─ results/v1/<repo>/<sha>/<cluster>/<test-pack>/      ← NEW (SHA-keyed verdicts)
       └─ <test-name>.json
  └─ results/v1/risk/<repo>/<sha>/<cluster>.json         ← NEW (risk-assessor output, see risk-assessor.md)
```

Each result file:

```json
{
  "schema_version": "v1",
  "test_name": "broker-registration-da",
  "test_pack": "playwright-ui",
  "service": "leartech-broker-ui",
  "sha": "abc123def",
  "cluster": "gcp",
  "status": "Success",
  "duration_ms": 12450,
  "started_at": "2026-04-29T14:23:01Z",
  "finished_at": "2026-04-29T14:23:14Z",
  "trace_url": "https://trace.playwright.dev/?trace=...",
  "trace_har_blob": "gs://leartech-qa-har/v1/<run-id>/trace.har",
  "metadata": {
    "playwright_version": "1.56.0",
    "browser": "firefox"
  }
}
```

The shift-left-tests quill query: "for service X, SHA Y, do all required tests have at least one `Success` entry across both clusters?" — a couple of GCS list calls + a JOIN with `required-tests/<service>.yaml`. Sub-second.

**Why GCS confirmed**:
- Bucket + auth pattern already exist; reuse not reinvent
- Append-only, immutable artifacts — natural fit for object storage
- Cheap storage at scale
- Easy to re-export / mirror for analysis
- Bucket lifecycle policy auto-expires old results (e.g. 90 days)

**Implementation**: extend `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml` to upload its existing `results.json` to the SHA-keyed path alongside the existing PR-keyed artifact uploads. ~10 added lines, ~2 hours of work — not the "1 day for new bucket + writer" originally estimated. Same pattern duplicates into other test-pack tasks (contract, integration, etc.) as they land — ~30 min each once the helper snippet is proven.

Counter-argument for Mongo: richer query capability for dashboards. Compromise: GCS as source of truth + a thin indexer that materializes recent results into Mongo for the dashboard. Defer until needed.

## Multi-cluster gate logic

Leartech runs both `tf-jx-usable-bird` (GCP) and `modern-burro` (Azure). Tests run on both. Gate must consult both.

```go
// Pseudocode for shift-left-tests quill
for _, requiredTest := range required {
    gcpResult := store.Latest(repo, sha, "gcp", requiredTest)
    azResult  := store.Latest(repo, sha, "az",  requiredTest)
    
    switch policy {
    case "both-must-pass":      // strictest; default
        if !gcpResult.Success || !azResult.Success { return Fail }
    case "either-must-pass":    // for tests that only run on one cluster
        if !gcpResult.Success && !azResult.Success { return Fail }
    case "primary-only":        // for tests that genuinely should only run on a primary cluster
        if !gcpResult.Success { return Fail }
    }
}
```

The policy per-test (or per-suite) is in `required-tests/<service>.yaml`. Default `both-must-pass` for everything new; specify exceptions explicitly.

This also incidentally helps detect cluster-divergence — a test that consistently passes on GCP but fails on Azure flags a real cluster-config drift.

## Override semantics

Lighthouse `override` plugin enabled for the leartech production GitOps repo. Comment `/override leartech-gate` to bypass the failing check.

**Audit**: every `/override` event hooks into a webhook (Lighthouse webhook plugin) that appends to `gate-metadata/overrides.log.jsonl` in qa-management:

```jsonl
{"ts":"2026-04-29T14:30:00Z","pr":12345,"repo":"JX3_Leartech_Production","actor":"alice","check":"leartech-gate","quill":"shift-left-tests","reason":null,"qa_management_tag":"v1.42.0"}
```

Weekly report from this log:
- Override frequency per quill (high frequency = quill is too strict / drift)
- Override actors (concentration in one person = single point of judgment)
- Reasons (if `/override` includes a reason — Lighthouse supports `/override <name> reason`)

This is the operational signal that's missing in mqube. If overrides are concentrated on one quill, that quill needs work, not more overrides.

**Approval policy**:

```yaml
# gate-metadata/overrides-policy.yaml
quills:
  shift-left-tests:
    blocking: true
    require_approver_from: codeowners-of-source-repo
  contract-tests:
    blocking: true
    require_approver_from: codeowners-of-source-repo
  load-sla:
    blocking: false                 # alert-only, no approver needed
```

Approver-required overrides on blocking quills require a separate `/approve override` comment from a different actor — Lighthouse can enforce this. Defaults to two-key.

**Risk-driven override stiffening** — when `risk-assessor` returns `risk_level: high` for the SHA, the gate reads `override_policy` from the assessment and elevates approval requirements automatically. See `risk-assessor.md` for the schema. Stops developers from `/override`-ing high-risk PRs without a second pair of eyes; doesn't penalize trivial PRs.

## `post-deploy-tests` quill (Phase 2.7)

Reads `gs://test-artifacts-product-first/results/v1/post-deploy/<repo>/<sha>/<cluster>/...` for results produced by `leartech-arrivals-observer` (see `arrivals-observer.md`). The arrivals-observer runs the same Playwright suite against staging-deployed versions; this quill verifies those staging tests passed before letting the gate clear.

**This quill closes a real gap in pre-merge-only testing**: preview env ≠ staging, so issues that only manifest with real staging config / scale / data are missed by shift-left. The quill catches them at the staging→prod hop.

```yaml
post-deploy-tests:
  blocking: false                # alert-only initially; promote to blocking after baseline period
  description: "Required Playwright tests passed against the staging deployment of the promoted version"
  impl: result-store-lookup
  config:
    result_store: gs://test-artifacts-product-first/results/v1/post-deploy/
    tolerate_missing_window: true   # if arrivals-observer hasn't observed yet, don't fail
```

**Quill behavior**:

```
For each promoted service v_new:
  Query result-store for results/v1/post-deploy/<service>/<sha>/...
  If no results found:
    if tolerate_missing_window: pass (with warning in PR comment)
    else: fail
  If results found:
    All required tests passed in window? → pass
    Otherwise → fail (newly-failed list rendered in PR comment)
```

**Known gap — an in-progress arrival reads as `fail` (observed 2026-07-07).** The `Otherwise → fail` branch lumps together two very different states: a *finalized* failure, and a promotion whose arrival is still `phase="Testing"` (the observer simply hasn't finished testing it yet). When a promotion lands **faster than the observer finishes testing it**, the gate goes **red for a false reason** and self-heals minutes later once the arrival finalizes to `Passed`. Fix: branch on **arrival phase** before the pass/fail decision — `Passed → pass`, `Failed → fail`, **`Testing`/`Pending` → pending (wait, don't block)** — leaving `tolerate_missing_window` to cover the genuinely-no-results case as today. Only a *finalized* `Failed` should block. This removes a whole class of transient false-reds. Real example: `leartech-auth-admin-ui 0.0.12` blocked the AZ gate as ❌ while its e2e was mid-run; the e2e passed ~40 min later and a re-run would have been green.

**Phased rollout**:

- **Weeks 1-6 of operation**: `blocking: false` — emits PR comment with results but does not fail the check. Tunes flake rates and forensics thresholds without disrupting releases.
- **Weeks 6-8**: per-service flip to `blocking: true` for services where the signal is reliable (override-rate stable, forensics consistently meaningful).
- **Steady state**: most production-critical services blocking; opt-in alert-only for less critical / flaky services.

This is meaningfully more conservative than mqube's behavior (their porcupine testing quill is blocking from day one). We get the safety valve while we calibrate.

**Why a separate quill from `shift-left-tests`**: distinguishing "pre-merge tests passed" (caught most things) from "post-deploy tests passed" (caught environmental + scale + cross-service issues) is operationally useful. Override audit can identify which class of failure dominates over time and inform what to invest in.

## Performance

The whole gate evaluation should run in **under 30 seconds** on a typical promotion PR (10-20 services changing). Budgets:

- Helmfile diff: <1s
- Read qa-management config: <2s (cached once at startup)
- Per-quill: <2s (small number of GCS lookups)
- PR comment composition + post: <2s

If gate runs >60s, something's wrong (probably result-store query inefficiency). Monitor as a Prometheus metric.

## What it doesn't do

- **Doesn't run tests.** Tests run pre-merge (shift-left) or scheduled (load-test cron); gate just consults verdicts.
- **Doesn't store anything mutable.** Read-only of qa-management + result-store. Writes only the audit log entry on override.
- **Doesn't talk to a fat-controller-equivalent.** No need for one in this design — shift-left is the test orchestration layer.
- **Doesn't post Slack.** Slack-alert layer is a separate, optional Phase-3 component (just reads the result-store + override log on a schedule).

## Build estimate (with leverage)

| Piece | Estimate | Reuses |
|---|---|---|
| Project skeleton from leartech-go-service-template | half day | golden template |
| Helmfile parse via beaver lib | 1 day | mqube's beaver is open / reusable |
| Result-store client (GCS read) | **~2 hours** | bucket + auth already provisioned by `end2end-ui` task |
| Three quill impls (ResultStoreLookup, HelmfileDiff base case) | 1.5 days | data-driven via config |
| Tekton task + Lighthouse trigger wiring | half day | leartech-pipeline-catalog `uses:` pattern |
| GitHub PR comment + status post | half day | reuse from mqube porcupine if license-compatible |
| Tests (unit + integration via preview-shift-left) | 1 day | preview-shift-left harness |
| Extend `end2end-ui` task to upload `results.json` to SHA-keyed path | ~2 hours | task already uploads to GCS |

Total: **~4.5 days** with foundation leverage. Without leverage (greenfield Go, no templates, no catalog, no bucket): ~2-3 weeks.

## Phase-out path for added quills

When new quill types are needed beyond the three impls (`ResultStoreLookup`, `HelmfileDiff`, `CronJobHealth`), the pattern is:

1. Add a new impl struct in `internal/quills/<name>/`
2. Add an entry in `quills.yaml` schema (CUE) for the new impl's config shape
3. Wire the new impl into the dispatch table in `cmd/api/main.go`
4. Cut a new `leartech-gate` image tag

Each new impl is ~1 day (assuming standard inputs/outputs). Marginal cost is low because the framework is settled — exactly the foundation-leverage compounding.
