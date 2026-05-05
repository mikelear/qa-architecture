# Session 0 — End-to-end spike brief

The first build session. **Goal: prove the architecture works end-to-end, low-fidelity, in one day.** Everything Phase 1 + later phases would build is deferred. Phase 1 sessions become hardening passes on top of what this spike produces.

This brief is designed to be read cold by a session starting fresh. Don't go off-script — every "while I'm here" addition costs the spike's value.

---

## Outcome we're proving

The spike uses **two new dedicated test fixtures** instead of leartech-auth-ui — own the fixture, own the failure modes, no risk to real services. Both are golden-template-derived, both are demo assets going forward.

- `mikelear/leartech-qa-canary` — Go service from `leartech-go-service-template`. Real `.lighthouse` triggers, real preview env, real Playwright + contract tests. Just exists for QA system validation.
- `mikelear/leartech-qa-sandbox-gitops` — minimal GitOps repo (helmfile + lighthouse triggers) including the canary. Promotion PRs target this; nothing real depends on it.

```
PR opened on leartech-qa-canary
        ↓
shift-left tests run (existing end2end-ui Tekton task)
        ↓
results.json lands in GCS at SHA-keyed path  ← NEW
        ↓
synthetic promotion PR opens against leartech-qa-sandbox-gitops
        ↓
leartech-gate Tekton presubmit fires  ← NEW
        ↓
gate reads results from GCS, parses helmfile via beaver, evaluates one quill
        ↓
GitHub PR check: green if results pass, red if missing/failing
        ↓
/override leartech-gate flips to passed
        ↓
PR mergeable
```

End-to-end run from PR open to gate verdict: target **<5 minutes**.

The canary supports three demo modes used in validation:

| Mode | What's deliberately wrong | What we expect to verify |
|---|---|---|
| **Happy** | Nothing — all tests green | Gate green; promotion PR mergeable |
| **Failure** | One Playwright test asserts impossible state | Gate red with named failing test in PR comment; `/override` works |
| **Missing results** | Manually delete results from GCS after they were uploaded | Gate red with "no results found in window" |

(A fourth mode — **Network**, where the canary calls a deliberately-broken downstream — is reserved for Phase 2.7 traffic-forensics validation. Out of scope for Session 0.)

---

## Pre-flight checklist

Foundation must already be in place before starting. If any of these are missing, fix THAT first; do not work around it.

- [ ] `gh auth status` shows authenticated as `mikelear`
- [ ] `~/leartech/leartech-go-service-template/` exists and is current
- [ ] `~/leartech/leartech-pipeline-catalog/` clean working tree on `main`
- [ ] GCS bucket `gs://test-artifacts-product-first/` accessible from cluster (existing — verified by `end2end-ui` task)
- [ ] `test-artifacts-gcs-key` Kubernetes secret in `jx` namespace (existing — verified by `end2end-ui` task)
- [ ] Lighthouse + Tide running on at least the GCP cluster (`tf-jx-usable-bird` per multi-cluster-jx3-pattern)
- [ ] `gh repo create` permissions on `mikelear/` org (we'll create two new repos)
- [ ] `yq eval` and `yamllint` available locally (used 2026-05-04 to catch the inline-Python trap)

---

## Deliverables

### 1. `mikelear/leartech-qa-management` repo created

Public, matching `mikelear/preview-shift-left` precedent. Minimal contents:

```
leartech-qa-management/
  README.md          — purpose, link to qa-architecture design docs
  CLAUDE.md          — orientation
  required-tests/
    leartech-auth-ui.yaml   — single service, hand-rolled YAML (no CUE yet)
  gate-metadata/
    quills.yaml             — only shift-left-tests entry
  OWNERS
  .gitignore
```

**Skip in this session**:
- CUE schemas
- Cross-reference linter / `make validate`
- Self-CI (Lighthouse triggers on this repo)
- Helm chart for downstream consumers
- `repo-type-policy/`, `service-catalog.yaml`, `risk-config.yaml`, `load/`, `test-catalog/`

### 2. Result-store path extension on `end2end-ui`

Modify `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml`. Add ~10 lines that, after the existing artifact upload, also upload the `results.json` to a SHA-keyed path:

```
gs://test-artifacts-product-first/results/v1/<repo>/<sha>/<cluster>/playwright-ui/<test-name>.json
```

Reuse the existing `gsutil` + `test-artifacts-gcs-key` auth pattern. Do not create a new bucket; do not refactor the artifact-upload logic.

**Skip**:
- Result-store helper extraction to `leartech-go-common`
- Schema versioning enforcement beyond the literal `"schema_version": "v1"` field
- Extending other test packs (contract, integration, etc.) — auth-ui's Playwright suite only

### 3. `mikelear/leartech-gate` Go binary

Init from `leartech-go-service-template`. Single binary, single Tekton task wrapper.

**Code structure (minimal):**

```
leartech-gate/
  cmd/gate/main.go            — argparse, calls into internal/
  internal/
    helmfile/                 — beaver lib import; parses promotion-PR helmfile diff
    resultstore/              — GCS read; one method: Latest(repo, sha, cluster, testPack, testName)
    quills/
      shiftleft/usecase.go    — single quill impl; reads required-tests YAML + queries result-store
    github/                   — PR comment + check status post
  charts/leartech-gate/       — defer; not deployed in this spike
  Dockerfile
  go.mod
```

**Behavior:**

1. Parse promotion-PR helmfile via beaver lib — get list of services + versions being promoted
2. For each service, fetch `required-tests/<service>.yaml` from `leartech-qa-management` (read raw via GitHub API in spike; later via Renovate-pinned tag)
3. For each required test, query GCS for `results.json` matching `(repo, sha, cluster, testPack, testName)`
4. Aggregate verdict: pass if all required tests have a `Success` result; fail otherwise
5. Post GitHub PR check via existing GitHub App credentials (whatever is on the cluster's tekton-bot)
6. Exit 0 (pass) or 1 (fail) so Lighthouse picks up the check status

**Skip**:
- Multi-quill framework — single quill is enough
- Multi-cluster query (just gcp; az results are out-of-scope for this run)
- Audit log of overrides
- Risk-driven override stiffening
- Cross-reference linter integration
- Refresh-on-tag-bump for qa-management (just read raw `@main` for spike)
- Tests beyond a single happy-path integration test

### 4. Tekton task wrapper

Add `tasks/qa-gate/pullrequest.yaml` to `leartech-pipeline-catalog`. Pattern: `uses:` your gate image at `:latest` (single image; tag advancement is later).

Validate with `yq eval` + `yamllint` before pushing — the new shared rule applies here.

### 5. Lighthouse trigger on a sandbox helmfile repo

Add `lint`-style trigger entry pointing at the new `qa-gate/pullrequest.yaml` source. **Use a test/sandbox helmfile repo OR a feature branch on a real GitOps repo** — do not gate real promotions on Day 1.

### 6. Four-mode end-to-end demo

Three runs against the canary + sandbox, each verifying a specific behavior. See **Appendix: Concrete test commands** below for the exact commands.

**Run 1 — Happy path**: PR on canary → end2end-ui green → results.json in GCS → synthetic promotion PR → gate fires → check is GREEN → PR mergeable.

**Run 2 — Failure path**: deliberately break a Playwright test on canary → push → end2end-ui fails → results.json shows status: Failure → new promotion PR → gate fires → check is RED with named failed test in PR comment.

**Run 3 — Override path**: continuing from Run 2's PR → comment `/override leartech-gate` → check flips to passed → PR mergeable.

**Run 4 — Missing-results path**: manually delete the result file from GCS → re-fire gate (empty commit on sandbox PR) → check is RED with "no results found in window".

If all four runs complete with expected outcomes, **Session 0 is done.**

---

## Validation criteria (definition of done)

All must be ticked:

- [ ] `mikelear/leartech-qa-management` published, public, with `required-tests/leartech-auth-ui.yaml` populated
- [ ] `mikelear/leartech-gate` published, builds clean, image `:latest` published to ghcr.io or ACR
- [ ] `end2end-ui` task uploads `results.json` to SHA-keyed GCS path on next auth-ui PR run
- [ ] Sandbox helmfile repo has Lighthouse trigger registered for `leartech-gate`
- [ ] Synthetic promotion PR fires `leartech-gate` check
- [ ] Check is green when results pass; red when missing
- [ ] `/override leartech-gate` works
- [ ] End-to-end takes <5 min from PR open to verdict (target; not blocker)
- [ ] No real production GitOps repo is gated yet — sandbox/test only
- [ ] `~/leartech/qa-architecture/sessions.md` live-status updated for Session 0
- [ ] Hardening backlog written into Phase 1 session briefs based on what felt fragile during the spike

---

## What we expect to break (and what to do about it)

These are predictable; account for them in the time budget rather than treating each as a blocker.

| Likely failure | Cause | Mitigation in spike |
|---|---|---|
| GCS path mismatch — gate reads where task didn't write | Naming spec drift between two new code paths | Hard-code the same path constant in both; centralize in Phase 1 |
| Beaver lib helmfile parse semantics | Library expectations vs leartech's helmfile shape | Use mqube-pipeline-catalog as reference; read its tests |
| GitHub App permissions for PR check post | Service-account scope is correct on paper, often wrong on first run | Reuse existing `tekton-bot` app token; do not create a new App |
| Lighthouse trigger registration timing | New triggers sometimes need a cluster-side reconcile | Wait 60s after pushing source-config; if still missing, kubectl describe sourcerepo to see git-operator status |
| Image push permissions | First push to ghcr.io/mikelear or ACR | Reuse the same pattern as `leartech-ai-classifier` |
| YAML parse on the new `qa-gate/pullrequest.yaml` | Inline scripts repeating 2026-05-01 trap | Pre-flight with `yq eval` + `yamllint`. The new lint presubmit on pipeline-catalog (f32a773) will catch it on PR |

---

## Anti-scope-creep checklist

Things that will tempt you. **Don't.**

- "While I'm here, let me add CUE schemas to qa-management" → **No.** Phase 1 hardening session.
- "Let me add a second quill so the framework feels real" → **No.** One quill proves the data-driven approach. Adding a second tempts you into refactoring before you know the first one's right.
- "Let me wire to a real GitOps repo for the demo" → **No.** Sandbox first. Real wiring is Phase 1 session 1.5.
- "Let me write integration tests" → **One** happy-path integration test is fine. Don't over-test before architecture is settled.
- "Let me add multi-cluster support" → **No.** GCP-only. Az parity in Phase 1.
- "Let me add a Helm chart for the gate" → **No.** Spike runs as a Tekton-task-invoked binary. Chart is Phase 1.
- "Let me add the override audit log" → **No.** Lighthouse's built-in plugin is enough for the spike. Audit log is Phase 1.
- "Let me populate `required-tests/` for more services" → **No.** Auth-ui only. Bulk seed is session 1.6.

If something feels load-bearing for the spike's success and is on this list, that's a sign of a missing pre-condition; pause and reconsider rather than scope-creep.

---

## Time budget

| Phase | Time |
|---|---|
| Foundation pre-flight | 30 min |
| qa-management repo + first required-tests file | 45 min |
| end2end-ui task extension | 30 min |
| leartech-gate Go binary skeleton + helmfile parse | 2 h |
| Single quill impl + result-store query | 1.5 h |
| GitHub check post + PR comment | 45 min |
| Tekton task wrapper + image build + push | 1 h |
| Lighthouse trigger registration | 30 min |
| End-to-end demo run + iterate on issues | 2 h |
| Live-status + hardening-backlog write-up | 30 min |
| **Total** | **~10 h** |

If you're at 8 hours and the demo run hasn't happened yet, **stop adding things and start integrating.** The first end-to-end run is more valuable than any individual component.

---

## After Session 0

Update `sessions.md` live-status. Each Phase 1 session's brief should reference the spike's commit SHAs and explicitly list what needs hardening based on what felt fragile. Don't write the hardening session briefs in advance — let the spike inform what's actually needed.

Likely Phase 1 reframing:

- **1.1 (was: stand up qa-management)** → "harden qa-management — add CUE schemas + cross-reference linter + auto-merge policy"
- **1.2** → "extend result-store path to other test packs (was just Playwright in spike)"
- **1.3-1.4** → "second + third quill in gate; extract data-driven config from hard-coded service list"
- **1.5** → "wire to real GitOps repos (was sandbox in spike)"
- **1.6** → "bulk repo-type.yaml seed across remaining repos"
- **1.7** → retro

Each becomes ~1-2 hours of focused hardening, not 4-5 hours of build-from-scratch.

---

## Reading order before starting

1. `~/leartech/qa-architecture/README.md` — exec summary
2. `~/leartech/qa-architecture/qa-management-repo.md` — schema you're partly implementing
3. `~/leartech/qa-architecture/gate.md` — gate design you're partly implementing
4. `~/leartech/Qa-Analysis/findings/06-porcupine.md` — mqube reference for the gate pattern
5. This brief — for what's IN and what's OUT
6. `~/leartech/hub/shared-rules/no-inline-python-in-tekton.md` — the trap you must not fall into

That's it. Get coding.

---

## Appendix: Concrete test commands

Copy-paste verification commands per step. Use these as definition-of-done evidence.

### Pre-flight

```bash
gh auth status                                       # mikelear authenticated
ls ~/leartech/leartech-go-service-template          # template exists
yq --version && yamllint --version                   # both tools available
gcloud storage ls gs://test-artifacts-product-first/ # bucket reachable
```

### After Step 1 — canary + sandbox

```bash
gh repo view mikelear/leartech-qa-canary
gh repo view mikelear/leartech-qa-sandbox-gitops

cd ~/leartech/leartech-qa-canary
make build
yq eval '.' .lighthouse/jenkins-x/triggers.yaml >/dev/null && echo OK
```

### After Step 2 — qa-management

```bash
gh repo view mikelear/leartech-qa-management

# required-tests parses
gh api repos/mikelear/leartech-qa-management/contents/required-tests/leartech-qa-canary.yaml \
  --jq .content -r | base64 -d | yq eval '.' >/dev/null && echo OK
```

### After Step 3 — result-store extension

```bash
# Push empty commit on a canary PR
git commit --allow-empty -m "test: verify result-store upload" && git push

# Wait for end2end-ui to complete, then verify GCS path exists
SHA=$(git rev-parse HEAD)
gcloud storage ls gs://test-artifacts-product-first/results/v1/leartech-qa-canary/$SHA/gcp/playwright-ui/

# Inspect content
gcloud storage cat gs://test-artifacts-product-first/results/v1/leartech-qa-canary/$SHA/gcp/playwright-ui/01-page-loads.json | jq .
# Expect: {schema_version: v1, status: Success, sha: <SHA>, ...}
```

### After Step 4 — leartech-gate

```bash
cd ~/leartech/leartech-gate
go test ./internal/quills/shiftleft/...     # unit tests pass
docker build -t leartech-gate:dev .
docker run leartech-gate:dev --help          # shows CLI args
```

### After Step 5 — Tekton task + Lighthouse trigger

```bash
cd ~/leartech/leartech-pipeline-catalog
yq eval '.' tasks/qa-gate/pullrequest.yaml >/dev/null && echo OK
yamllint tasks/qa-gate/pullrequest.yaml      # clean

kubectl describe sourcerepo leartech-qa-sandbox-gitops -n jx 2>&1 | grep -i webhook
```

### Step 6 — Run 1 (Happy)

```bash
# 1. Push happy-path commit on canary PR
cd ~/leartech/leartech-qa-canary
git commit --allow-empty -m "test: happy path" && git push
SHA=$(git rev-parse HEAD)

# 2. Wait for end2end-ui green; results.json in GCS
~/leartech/hub/scripts/pr-pipelines.sh leartech-qa-canary <CANARY_PR>

# 3. Open synthetic promotion PR against sandbox
cd ~/leartech/leartech-qa-sandbox-gitops
git checkout -b promote-canary-$SHA
yq -i ".releases[] |= select(.name == \"leartech-qa-canary\") .version = \"$SHA\"" \
  helmfiles/staging/helmfile.yaml
git commit -am "promote: leartech-qa-canary $SHA" && git push
gh pr create --title "promote canary $SHA" --body "spike test run 1: happy"

# 4. Verify gate fires + green
SANDBOX_PR=$(gh pr list --head promote-canary-$SHA --json number -q '.[0].number')
gh pr checks $SANDBOX_PR | grep leartech-gate    # expect: pass
gh pr view $SANDBOX_PR                            # expect: PR comment from gate with verdict
```

### Step 6 — Run 2 (Failure)

```bash
# 1. Break a Playwright test on canary
cd ~/leartech/leartech-qa-canary
# edit end2end-ui/01-page-loads.spec.ts: change `expect(true).toBe(true)` to `expect(true).toBe(false)`
git commit -am "fail: deliberate test break" && git push
FAIL_SHA=$(git rev-parse HEAD)

# 2. Wait for end2end-ui to fail
~/leartech/hub/scripts/pr-pipelines.sh leartech-qa-canary <CANARY_PR> | grep end2end-ui
gcloud storage cat gs://test-artifacts-product-first/results/v1/leartech-qa-canary/$FAIL_SHA/gcp/playwright-ui/01-page-loads.json | jq .status
# Expect: "Failure"

# 3. New promotion PR for failed SHA
cd ~/leartech/leartech-qa-sandbox-gitops
git checkout -b promote-canary-$FAIL_SHA
yq -i ".releases[] |= select(.name == \"leartech-qa-canary\") .version = \"$FAIL_SHA\"" \
  helmfiles/staging/helmfile.yaml
git commit -am "promote: leartech-qa-canary $FAIL_SHA" && git push
gh pr create --title "promote canary $FAIL_SHA" --body "spike test run 2: failure"

# 4. Verify gate fires + RED
FAIL_SANDBOX_PR=$(gh pr list --head promote-canary-$FAIL_SHA --json number -q '.[0].number')
gh pr checks $FAIL_SANDBOX_PR | grep leartech-gate    # expect: fail
gh pr view $FAIL_SANDBOX_PR                            # expect: PR comment listing failed test
```

### Step 6 — Run 3 (Override)

```bash
# Continuing from Run 2's PR
gh pr comment $FAIL_SANDBOX_PR --body "/override leartech-gate"
sleep 10
gh pr checks $FAIL_SANDBOX_PR | grep leartech-gate    # expect: pass (overridden)
```

### Step 6 — Run 4 (Missing-results)

```bash
# Delete result file from GCS
gcloud storage rm gs://test-artifacts-product-first/results/v1/leartech-qa-canary/$FAIL_SHA/gcp/playwright-ui/01-page-loads.json

# Re-fire gate by pushing empty commit on sandbox PR
git checkout promote-canary-$FAIL_SHA
git commit --allow-empty -m "retrigger gate" && git push

# Verify gate fails with "results missing in window"
sleep 60
gh pr checks $FAIL_SANDBOX_PR | grep leartech-gate    # expect: fail
gh pr view $FAIL_SANDBOX_PR                            # expect: comment "no results found for SHA"
```

### Cleanup after spike

```bash
# Close test PRs (don't merge anything)
gh pr close $SANDBOX_PR        # happy
gh pr close $FAIL_SANDBOX_PR   # failure/override/missing
# Canary PR stays open if it's a long-lived test fixture branch

# Lessons + hardening backlog → update sessions.md live-status
```
