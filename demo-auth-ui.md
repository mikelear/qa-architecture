# End-to-end QA system demo — `leartech-auth-ui` (Playwright)

A walkthrough of how a single PR moves through the leartech QA stack, end-to-end. Use `leartech-auth-ui` because it already has 7 Playwright specs covering login, 2FA, token renewal, sign-out, and the about page — the test pack that gives the richest diagnostic output.

Audience: anyone seeing the QA system for the first time. Read top-to-bottom; each stage is a concrete artifact you can point at in a live demo.

## Prerequisites for the demo

| What | Why |
|---|---|
| `leartech-auth-ui` repo | Has `end2end-ui/` with Playwright specs |
| Two GitOps repos: `jx-build-cluster-{gsm,akv}` | Shows multi-cluster gate verdict |
| `tekton-git` ExternalSecret in `jx-staging` (both clusters) | Job runner clones the (private) PR repo |
| `arrivals-observer` 0.0.13+ deployed | Watcher fallback + GIT_TOKEN env active |
| `qa-gate` Lighthouse trigger registered, `optional: false` | Gate actually blocks |

All ✅ in place as of 2026-05-08.

## Stage 1 — A developer opens a PR

Engineer pushes a commit to a feature branch; opens PR.

```
$ gh pr create --title "fix(login): handle empty TOTP gracefully"
```

Lighthouse fires the `auth-ui` repo's 11 presubmit checks in parallel. Two are relevant to this demo:

- `verify` — chart lint + helmfile dry-run
- `end2end-ui` — Playwright pack against the PR's preview environment

Inspectable in **Tekton dashboard** (`https://tekton-dashboard-jx.jx.leartech.com`) or via PR check links on GitHub.

## Stage 2 — Preview environment

JX deploys the PR's chart to a per-PR namespace:

```
$ kubectl get ns | grep auth-ui-pr
jx-mikelear-leartech-auth-ui-pr-39   Active   2m
```

URL: `https://leartech-auth-ui-pr-39.jx.leartech.com` (auto-DNS via externalDNS + cert-manager).

This is the **shift-left** environment. The PR's specific code, isolated.

## Stage 3 — `end2end-ui` runs against preview

The Tekton task `end2end-ui` (catalog: `mikelear/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml`) does:

1. Wait for `<preview-url>/health/live` to return 200 three times in a row
2. `npm ci` + `npx playwright test --reporter=list,html`
3. `--trace=on --screenshot=on --video=retain-on-failure` so failed runs have a click-through artifact set
4. Upload to `gs://test-artifacts-product-first/<repo>/pr-39/<cluster>/<ts>/` (PR-keyed) AND
5. Upload `results.json` to `gs://test-artifacts-product-first/results/v1/<repo>/<sha>/<cluster>/end2end-ui/results.json` (SHA-keyed — the **shift-left result store**)

If a spec fails:
- PR sticky comment: ":x: end2end-ui: FAIL — 03-login-flow.spec.ts (login redirect timeout)"
- Comment includes a `[trace](https://trace.playwright.dev/?trace=<signed-url>)` link
- `end2end-ui` GitHub check goes red → branch protection blocks merge

Engineer fixes the bug; pushes again; cycle repeats until green.

## Stage 4 — PR merges → release pipeline → auto-promotion

PR green → tide auto-merges (or human merge). Postsubmit fires:

1. `release.yaml` builds the image: `us-central1-docker.pkg.dev/product-first/oci/leartech-auth-ui:0.0.37`
2. Helm chart packaged + pushed to `oci-charts/leartech-auth-ui:0.0.37`
3. `jx-promote` opens an auto-promotion PR on **both** `jx-build-cluster-gsm` and `jx-build-cluster-akv` GitOps repos:

```
PR #287 on jx-build-cluster-gsm:
  + version: 0.0.37   (was: 0.0.36)
   in helmfiles/jx-staging/helmfile.yaml under leartech-auth-ui
```

## Stage 5 — Gate fires on the auto-promotion PR

Lighthouse's `qa-gate` presubmit fires (registered today, `optional: false`). Tekton runs `gate-cli` which:

1. Reads the PR's helmfile → 24 services (auth-ui, auth-service, canary, …)
2. For each, evaluates **two quills**:

```
shift-left-tests quill (per service)
  → does qa-management/required-tests/<svc>.yaml exist?
  → for each required test, GCS lookup at
    gs://test-artifacts/results/v1/<repo>/<sha>/<cluster>/<pack>/<test>.json
  → all Status=Success? → pass  | any missing/Failed → fail

post-deploy-tests quill (per service)
  → kubectl-equivalent K8s API list of
    Arrival{label.qa.leartech.com/service=<svc>,version=<ver>} in jx-staging
  → phase=Passed → pass
  → phase=Skipped → pass-through (no testPacks configured)
  → phase=Failed/Timeout → fail
  → no Arrival → pass-through (not yet deployed)
```

3. Aggregated verdict → markdown table → sticky PR comment:

```
## :x: leartech-gate: FAIL

| Service               | Version | Verdict | Reason                                      |
|-----------------------|---------|---------|---------------------------------------------|
| leartech-auth-ui      | 0.0.37  | :x:     | shift-left: 1 missing; post-deploy: pending |
| ...                   | ...     | ...     |                                              |
```

If FAIL → `qa-gate` GitHub check fails → branch protection blocks merge.

## Stage 6 — Override path (if needed)

For legitimate bypasses (test flake, known issue, urgent ship):

```
/override leartech-gate
```

Lighthouse marks the check passing. PR mergeable. Override is permanent in the PR comment history; auditable.

## Stage 7 — auto-promotion PR merges → bootjob → staging deploys

Tide merges the GitOps PR. Postsubmit `bootjob` runs `jx gitops helmfile resolve` + `helmfile apply`. New ReplicaSet for auth-ui 0.0.37 lands in jx-staging.

```
$ kubectl get rs -n jx-staging | grep auth-ui
leartech-auth-ui-7c9d4f88b   2  2  2   18s
```

## Stage 8 — `arrivals-observer` notices

The watcher (running in jx-staging) sees the RS Add event:

```log
[INF] arrival recorded   arrival=leartech-auth-ui-0-0-37-jx-staging
                          rs=leartech-auth-ui-7c9d4f88b
                          service=leartech-auth-ui
                          version=0.0.37
```

Creates an `Arrival` CR:

```yaml
apiVersion: qa.leartech.com/v1alpha1
kind: Arrival
metadata:
  name: leartech-auth-ui-0-0-37-jx-staging
spec:
  service: leartech-auth-ui
  version: "0.0.37"
  replicaSet: leartech-auth-ui-7c9d4f88b
  deployedAt: "2026-05-08T15:30:01Z"
  stagingUrl: https://leartech-auth-ui-jx-staging.jx.leartech.com
  testPacks:
  - { name: end2end-ui, type: end2end-ui }
status:
  phase: Pending
```

## Stage 9 — Controller dispatches the test Job

The Arrival controller picks up `phase=Pending`:

```log
[INF] dispatching tests — Pending → Testing
      arrival=leartech-auth-ui-0-0-37-jx-staging
      stagingUrl=https://leartech-auth-ui-jx-staging.jx.leartech.com
      packs=1
```

Creates a K8s Job using the `playwright-runner` image:

```
$ kubectl get jobs -n jx-staging -l qa.leartech.com/job-kind!=forensics
ar-leartech-auth-ui-0-0-37-jx-staging-end2end-ui   Running   0/1   45s
```

The runner (inline bash inside the Job container):

1. Mounts `tekton-git/password` as `GIT_TOKEN` env
2. Mounts `test-artifacts-gcs-key` as the GCS upload credentials
3. Tries to clone `https://x-access-token:$GIT_TOKEN@github.com/mikelear/leartech-auth-ui.git` at `v0.0.37-gcp` → succeeds
4. Waits for `https://leartech-auth-ui-jx-staging.jx.leartech.com/health/live` to be healthy
5. `cd end2end-ui && npm ci && npx playwright test --trace=on --screenshot=on --video=retain-on-failure --reporter=list,html`
6. **Uploads to GCS:**
   - `gs://test-artifacts/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/results.json`
   - `gs://test-artifacts/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/test-results/` (trace.zip + screenshots + videos per failed test)
   - `gs://test-artifacts/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/playwright-report/index.html` (browseable report)

## Stage 10 — Arrival reaches a terminal phase

Job completes; controller polls and updates:

**Happy path** — Job exit 0 → `phase=Passed`:

```yaml
status:
  phase: Passed
  finalizedAt: "2026-05-08T15:34:18Z"
  tests:
  - name: end2end-ui
    type: end2end-ui
    status: Passed
    completedAt: "2026-05-08T15:34:18Z"
    jobName: ar-leartech-auth-ui-0-0-37-jx-staging-end2end-ui
```

**Sad path** — Job exit !=0 → `phase=Failed` AND controller fires the **forensics runner**:

```yaml
status:
  phase: Failed
  finalizedAt: "2026-05-08T15:34:18Z"
  tests:
  - name: end2end-ui
    status: Failed
    jobName: ar-leartech-auth-ui-0-0-37-jx-staging-end2end-ui
  forensics:
    jobName: forensics-leartech-auth-ui-0-0-37-jx-staging
    diffUrl: gs://test-artifacts/forensics/v1/leartech-auth-ui/0.0.37/gcp/diff.json
    generatedAt: "2026-05-08T15:34:42Z"
    summary:
      missing: 0
      new: 1
      latency_regressions: 2
      error_rate_regressions: 0
```

The forensics-runner queries Tempo for two windows around `deployedAt`:
- Before: `(deployedAt - 5min, deployedAt)` — last spans of the previous version (0.0.36, found via `findPreviousVersion`)
- After:  `(deployedAt, deployedAt + 5min)` — first spans of the new version

Computes per-endpoint regressions, attaches a sample `traceID` per regression, uploads diff.json.

## Stage 11 — Next promotion PR sees the verdict

When the next promotion PR opens (e.g. arrivals-observer 0.0.17 promo, or any service that includes auth-ui in its helmfile diff), the gate fires again. This time the post-deploy quill finds the Arrival:

```
| leartech-auth-ui | 0.0.37 | :x: | post-deploy: Arrival.phase=Failed; forensics: 2 latency regressions, 1 new endpoint ([diff](gs://test-artifacts/forensics/v1/leartech-auth-ui/0.0.37/gcp/diff.json)) |
```

Engineer clicks the diff link → JSON shows:

```json
{
  "latency_regressions": [
    {
      "endpoint": "/api/auth/totp/verify",
      "before": 180,
      "after": 1850,
      "ratio": 10.27,
      "sample_trace_id": "abc123def456",
      "trace_url": "http://tempo.jx-observability:3200/api/traces/abc123def456"
    }
  ]
}
```

Engineer clicks the trace URL → opens the actual span tree showing where TOTP verify slowed 10× — actionable in seconds.

## Stage 12 — All artifacts available for any test

```
$ gsutil ls gs://test-artifacts/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/
results.json
playwright-report/
  index.html             ← Playwright HTML report, full timeline + console
test-results/
  03-login-flow-spec/
    trace.zip            ← https://trace.playwright.dev/?trace=<signed-url>
    test-failed-1.png    ← screenshot at moment of failure
    video.webm           ← screen recording of the run
```

For a Playwright failure, the engineer's workflow:
1. Read PR comment row → "03-login-flow.spec.ts failed"
2. Click `[trace]` link → trace.playwright.dev opens with full timeline
3. See exact moment the redirect didn't happen, network panel, console, DOM snapshot
4. Compare with `screenshot` if visual regression
5. If perf-related, also click `[diff]` from the gate verdict to see Tempo span data

## Demo runbook (literal commands for a live walk-through)

```bash
# Setup — pick a representative PR (auth-ui has good Playwright coverage)
PR=39   # whatever the latest open PR is on auth-ui

# 1. Show the PR
gh pr view $PR --repo mikelear/leartech-auth-ui --web

# 2. Show the live preview
open https://leartech-auth-ui-pr-$PR.jx.leartech.com

# 3. Show Tekton tasks running
open https://tekton-dashboard-jx.jx.leartech.com/

# 4. Show shift-left results in GCS (after PR has e2e-ui presubmit run)
gsutil ls gs://test-artifacts-product-first/results/v1/leartech-auth-ui/

# 5. PR merges → show the auto-promotion PR
gh pr list --repo mikelear/jx-build-cluster-gsm --search "auth-ui in:title"

# 6. Show the gate verdict on that PR (PR sticky comment)
gh pr view 287 --repo mikelear/jx-build-cluster-gsm --comments | head -30

# 7. After auto-merge → show the live Arrival (phase progresses)
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get arrivals -n jx-staging --watch

# 8. Show the dispatched Job
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get jobs -n jx-staging -l qa.leartech.com/test-pack=end2end-ui

# 9. Watch the runner pod
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird logs -n jx-staging -l qa.leartech.com/test-pack=end2end-ui -f

# 10. Once Arrival is finalized, show the post-deploy artifacts
gsutil ls gs://test-artifacts-product-first/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/

# 11. Open the Playwright HTML report directly
open https://storage.googleapis.com/test-artifacts-product-first/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/playwright-report/index.html

# 12. If a test failed, open the trace in trace.playwright.dev
open "https://trace.playwright.dev/?trace=https://storage.googleapis.com/test-artifacts-product-first/results/v1/post-deploy/leartech-auth-ui/0.0.37/gcp/end2end-ui/test-results/03-login-flow-spec/trace.zip"

# 13. Show the forensics summary on Failed Arrivals
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get arrival leartech-auth-ui-0-0-37-jx-staging -n jx-staging -o jsonpath='{.status.forensics}' | jq

# 14. Open the diff.json
gsutil cat gs://test-artifacts-product-first/forensics/v1/leartech-auth-ui/0.0.37/gcp/diff.json | jq
```

## What's still missing for the full "wow factor" demo

| Gap | Effort | Effect |
|---|---|---|
| **CORS on test-artifacts bucket for trace.playwright.dev** | 10min terraform | Currently signed URLs work; without CORS, public trace viewer can't fetch. Same fix mqube uses on Azure Blob. |
| **Gate-cli adds direct Playwright trace links** | 1h gate-cli refactor | Today gate verdict has [diff] for forensics; should also have [trace] linking to playwright-report/index.html and the per-test trace.zip |
| **Result-store reader** (2.7.6) | 4-6h | Per-test entries in Arrival.status.tests[] (currently single pack-level entry). Better gate verdicts ("login-flow-spec failed" vs "end2end-ui Failed") |
| **Slack alerter** (2.7.4) | 2-3h | Push notifications when an Arrival fails, with diff URL inline |
| **Heading**: arrivals-observer-ui mirroring mqube-architect-ui | 1 week | Live dashboard. Defer until heavy use. |

## Why this matters

**Pre-merge testing alone catches PR-time bugs.** Things it can't catch:
- Environment-only secrets that work in preview but not staging
- Cross-service regressions (PR for service A passes A's tests; breaks B at runtime)
- Scale issues (preview is 1 pod; staging has real load)
- Data-shape mismatches (preview is synthetic; staging has accumulated real-world data)
- Dependency upgrades that pass unit tests but break against real downstream APIs
- Auth/Hydra OAuth flow misconfigs visible only against the real flow

The leartech QA stack catches all six in **post-deploy** — before promotion to production. With forensics + Playwright artifacts, diagnosis happens directly from the PR comment.
