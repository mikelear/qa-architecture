# Session 0 — End-to-end spike brief

The first build session. **Goal: prove the architecture works end-to-end, low-fidelity, in one day.** Everything Phase 1 + later phases would build is deferred. Phase 1 sessions become hardening passes on top of what this spike produces.

This brief is designed to be read cold by a session starting fresh. Don't go off-script — every "while I'm here" addition costs the spike's value.

---

## Outcome we're proving

```
PR opened on leartech-auth-ui (the canary service)
        ↓
shift-left tests run (existing end2end-ui Tekton task)
        ↓
results.json lands in GCS at SHA-keyed path  ← NEW
        ↓
synthetic promotion PR opens against test/sandbox helmfile
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

---

## Pre-flight checklist

Foundation must already be in place before starting. If any of these are missing, fix THAT first; do not work around it.

- [ ] `gh auth status` shows authenticated as `mikelear`
- [ ] `~/leartech/leartech-go-service-template/` exists and is current
- [ ] `~/leartech/leartech-pipeline-catalog/` clean working tree on `main`
- [ ] GCS bucket `gs://test-artifacts-product-first/` accessible from cluster (existing — verified by `end2end-ui` task)
- [ ] `test-artifacts-gcs-key` Kubernetes secret in `jx` namespace (existing — verified by `end2end-ui` task)
- [ ] Lighthouse + Tide running on at least the GCP cluster (`tf-jx-usable-bird` per multi-cluster-jx3-pattern)
- [ ] At least one open PR on `leartech-auth-ui` to use as the canary
- [ ] A test/sandbox GitOps repo OR a feature branch on a real one we won't merge — for the synthetic promotion PR
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

### 6. End-to-end demo run

Manual sequence to prove the spike:

1. Push a small commit to an open `leartech-auth-ui` PR (matching pattern from 2026-05-04 fix verification)
2. Wait for `end2end-ui` to complete; verify `results.json` appears at the new SHA-keyed GCS path
3. Open a synthetic promotion PR against the sandbox helmfile repo updating auth-ui's version to that SHA
4. Verify `leartech-gate` Lighthouse check appears on the PR
5. Verify the check is **green** because results exist and pass
6. Manually delete or rename the GCS results file
7. Re-trigger the gate (push empty commit or comment `/test leartech-gate`)
8. Verify the check is now **red**
9. Comment `/override leartech-gate` on the PR
10. Verify check transitions to passed; PR becomes mergeable

If steps 1-10 work, **Session 0 is done.**

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
