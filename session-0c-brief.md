# Session 0c — Close the cluster-side loop

The loose end from Session 0. Local gate-cli is fully working; the cluster-side flow couldn't be demonstrated because the gate's release pipeline aborted before pushing the image to a registry. This session closes that gap: image to registry, real PR on sandbox, qa-gate Lighthouse check fires + posts PR comment + sets verdict.

After this, the QA gate is genuinely closed-loop on the cluster — every step in the architecture has been exercised under real conditions.

**Time budget**: 1.5-2 hours.

---

## Outcome we're proving

```
PR opened on leartech-qa-sandbox-gitops bumping leartech-qa-canary's
   helmfile version to a SHA we have GCS results for
        ↓
Lighthouse fires qa-gate presubmit on both clusters
        ↓
qa-gate Tekton task pulls ghcr.io/mikelear/leartech-gate:latest
        ↓
gate-cli runs against the helmfile + qa-management raw + GCS
        ↓
PR check status: pass / fail
PR sticky comment: rendered verdict with per-test breakdown
        ↓
Optional: /override leartech-gate verifies the override path
        ↓
PR mergeable on green or override
```

End-to-end run from PR open to qa-gate verdict: target **<3 minutes** (smaller than Session 0 because the pipeline catalog tasks are minimal — git-clone-pr + gate-cli invocation, no preview-env, no scans).

---

## Pre-flight checklist

- [ ] `~/leartech/leartech-gate/` checked out clean (we left it on `main` after Session 0)
- [ ] `~/leartech/leartech-qa-sandbox-gitops/` checked out clean
- [ ] `gh auth status` — `mikelear` authenticated
- [ ] Local `gate-cli` binary build passes (`cd ~/leartech/leartech-gate && go build ./cmd/gate-cli`) — we know this works from Session 0
- [ ] Earlier demo data still in GCS at `gs://test-artifacts-product-first/results/v1/leartech-qa-canary/demosha-happy-path-v1/gcp/end2end/smoke.json` (or upload fresh — see Step 4)

---

## Deliverables

### Step 1 — diagnose openapi-generation failure

The gate's release pipeline failed with `openapi-generation` after `from-build-pack` succeeded. Diagnose first; the fix likely falls into one of two patterns:

**Pattern A: gate's swagger.json is broken** — golden Go template ships with swagger annotations + a generated swagger.json for an HTTP service. Gate is a CLI; the swagger probably doesn't match the actual gate-cli (no HTTP routes annotated). openapi-generation's parser may reject it.

**Pattern B: openapi-generation is unconditionally enabled** in the release pipeline and we just need to skip it for non-HTTP-service repos.

Investigation commands:

```bash
# Look at the actual failure log
~/leartech/hub/scripts/release-pipelines.sh leartech-gate --cluster az --failed-only --logs

# Look at the gate's swagger.json content
cat ~/leartech/leartech-gate/docs/swagger.json | head -20

# Look at how the release pipeline invokes openapi-generation
cat ~/leartech/leartech-gate/.lighthouse/jenkins-x/release.yaml
```

### Step 2 — fix or skip openapi-generation

Two implementation paths:

**Path A: keep cmd/server (HTTP service) + fix the swagger** — gate stays dual-purpose (CLI for Tekton invocation + HTTP server for chart deployment). swagger.json describes the /health endpoints. openapi-generation succeeds. ~30 min.

**Path B: remove openapi-generation step from gate's release.yaml** — gate has no API to generate clients for; the step is dead weight. Fastest fix. ~15 min.

Lean **Path B** for spike speed. Document as Phase 1 hardening: re-enable openapi-generation when gate's API surface (if any — maybe none) stabilizes.

### Step 3 — verify image push lands

After release pipeline runs cleanly, confirm the image is reachable:

```bash
# After triggering a fresh release (push a small commit on gate's main):
~/leartech/hub/scripts/release-pipelines.sh leartech-gate --cluster both
# Expect: SUCCESS on both clusters

# Verify image pullable:
docker pull ghcr.io/mikelear/leartech-gate:latest 2>&1 | tail -5
# OR if image is on a different registry, check that one
```

If image is on a different registry than what `tasks/qa-gate/pullrequest.yaml` references, **update the task** to point at the actual registry (commit + push to pipeline-catalog).

### Step 4 — ensure GCS results exist for the demo SHA

We have demo results from Session 0:
- `demosha-happy-path-v1` → smoke.json with status:Success
- `demosha-failure-path-v1` → smoke.json with status:Failure

Verify they're still there:

```bash
gcloud storage ls "gs://test-artifacts-product-first/results/v1/leartech-qa-canary/demosha-happy-path-v1/gcp/end2end/"
gcloud storage ls "gs://test-artifacts-product-first/results/v1/leartech-qa-canary/demosha-failure-path-v1/gcp/end2end/"
```

If they were lifecycle-cleaned, re-upload using the same fixtures from Session 0 (recipe in `session-0-brief.md` Appendix Step 6 Run 1 / Run 2).

### Step 5 — open real sandbox PR; verify qa-gate fires

Promotion PR on the sandbox repo bumping canary's version to the demo SHA:

```bash
cd ~/leartech/leartech-qa-sandbox-gitops
git checkout -b promote-canary-happy-demo
yq -i '.releases[] |= select(.name == "leartech-qa-canary") .version = "demosha-happy-path-v1"' helmfiles/staging/helmfile.yaml
git commit -am "promote: leartech-qa-canary demosha-happy-path-v1" 
git push -u origin promote-canary-happy-demo
gh pr create --title "Demo: happy-path qa-gate run" --body "Session 0c validation"
```

Watch the qa-gate Lighthouse check fire:

```bash
SANDBOX_PR=$(gh pr list --repo mikelear/leartech-qa-sandbox-gitops --head promote-canary-happy-demo --json number -q '.[0].number')
gh pr checks $SANDBOX_PR --repo mikelear/leartech-qa-sandbox-gitops
# Expect: gcp/qa-gate + az/qa-gate, both pass
```

Verify the PR sticky comment appears with the rendered verdict markdown.

### Step 6 — failure-path PR

Same pattern, different SHA:

```bash
git checkout -b promote-canary-failure-demo
yq -i '.releases[] |= select(.name == "leartech-qa-canary") .version = "demosha-failure-path-v1"' helmfiles/staging/helmfile.yaml
git commit -am "promote: leartech-qa-canary demosha-failure-path-v1"
git push -u origin promote-canary-failure-demo
gh pr create --title "Demo: failure-path qa-gate run" --body "Session 0c validation"
```

Verify qa-gate is RED + cited reason in PR comment.

### Step 7 — override demo on the failure PR

```bash
gh pr comment $FAIL_SANDBOX_PR --repo mikelear/leartech-qa-sandbox-gitops --body "/override qa-gate"
sleep 30
gh pr checks $FAIL_SANDBOX_PR --repo mikelear/leartech-qa-sandbox-gitops | grep qa-gate
# Expect: pass (Overridden by mikelear)
```

### Step 8 — capture lessons

Update `~/leartech/qa-architecture/session-0-lessons.md` with:

- Bootstrap gap #5 (if openapi-generation default-enabled is a real friction point for non-API services) or note its resolution
- Image-registry-and-task-reference timing (when do consumers see new `:latest`?)
- Total wall-clock from sandbox PR open to qa-gate verdict (target <3 min; capture actual)

---

## Validation criteria (definition of done)

- [ ] `ghcr.io/mikelear/leartech-gate:latest` (or wherever) reachable from both clusters
- [ ] gate's release pipeline ran SUCCESS on both clusters' main
- [ ] sandbox happy PR: `qa-gate` check is green; PR comment shows rendered verdict
- [ ] sandbox failure PR: `qa-gate` check is red; PR comment cites failed test
- [ ] `/override qa-gate` flips to passed
- [ ] `<3 min` end-to-end from PR open to qa-gate verdict
- [ ] `session-0-lessons.md` updated with anything new captured
- [ ] sessions.md live-status: Session 0c marked complete

---

## What we deliberately SKIP

- **Multi-quill framework** — single quill (shift-left-tests) stays. Phase 1.3.
- **CUE schemas on qa-management** — Phase 1.1.
- **Authenticated GCS reads** — public-bucket HTTPS works for the spike. Phase 1 hardening.
- **Risk-driven override stiffening** — Phase 2.5.
- **Auto-merging the demo PRs** — leave open + close at end of session. Real GitOps repos are NOT touched.

---

## Anti-scope-creep checklist

- "While I'm here, let me add the second quill (contract-tests)" → **No.** Phase 1.3 hardening session.
- "Let me wire qa-gate to a REAL GitOps repo (jx-build-cluster-gsm)" → **No.** Sandbox only. Real wiring is Phase 1.5.
- "Let me also do the basic-dependency quill" → **No.** Phase 1.4.
- "Let me write tests for the gate" → **One** smoke test fine. More is Phase 1.

---

## Anticipated failure modes

| Failure | Likely cause | Fix |
|---|---|---|
| openapi-generation still fails after Path B | Pipeline catalog wrapper unconditionally invokes the step | Investigate `release.yaml`'s `uses:` chain; may need to fork `from-build-pack` for the gate |
| Image not pullable from cluster despite SUCCESS | Image registry vs cluster pull-secret mismatch | Verify cluster's `imagePullSecrets` include the registry; maybe push to a different registry |
| qa-gate task can't pull tekton-git secret | Cluster auth scope | Add `optional: true` to env-var lookup; gate handles missing token gracefully |
| qa-gate Tekton runs but gate-cli exits non-zero unexpectedly | Helmfile path mismatch (sandbox repo vs the task's expected `helmfiles/staging/helmfile.yaml`) | Verify path matches what task config expects |
| GitHub PR comment fails with 403 | tekton-git secret scope | Validate token has issues:write |

Each is a candidate lesson for the runbook if hit.

---

## Reading order

1. `~/leartech/qa-architecture/sessions.md` (live-status section) — what's currently green/red
2. `~/leartech/qa-architecture/session-0-lessons.md` — bootstrap runbook so far
3. `~/leartech/qa-architecture/gate.md` — gate design
4. `~/leartech/leartech-pipeline-catalog/tasks/qa-gate/pullrequest.yaml` — the Tekton task we're firing
5. This brief

That's it. Get coding.
