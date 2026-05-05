# Session 0 — lessons learnt (live capture)

Running log of every friction point, gotcha, ordering dependency, and manual step encountered during the spike. Goal: extract a runbook the automated-agent can follow to bootstrap new repos through PR + release-to-staging without human intervention.

This file gets cleaned up + extracted at end of Session 0 into either:

- `~/leartech/hub/shared-rules/new-repo-bootstrap-runbook.md` (for human reference + AI review grounding), or
- A structured procedure in `~/leartech/automated-agent/` (for the agent's use)

Status: **live during spike — entries are timestamped and sometimes terse**.

---

## 0. Pre-flight

**2026-05-05 ~16:00** — verified before starting. Foundation in place:

- `gh auth status` — `mikelear` authenticated; token has `repo` + `delete_repo` scopes
- Golden template available at `~/leartech/leartech-go-service-template/`
- `yq v4.45.1`, `yamllint 1.36.2`, `go 1.25.4` all on PATH
- GCS bucket `gs://test-artifacts-product-first/` reachable (existing entries: `leartech-angular-service-template/`, `leartech-auth-ui/`, `webcoder-ui/`)
- `gsutil 5.35` working (note: this is `~/leartech/google-cloud-sdk/bin/gsutil`, not Homebrew)

**Lesson — pre-flight checklist as code**: the agent should run a `pre-flight.sh` that exits non-zero if any of these fail. Saves debugging mid-flow.

---

## 1. Repo creation

**Step**: `gh repo create mikelear/leartech-qa-canary --public --description "..."`

**What worked**:

- Public repo creation succeeds immediately under `mikelear` account; no review required
- Description is preserved on the GitHub UI
- `--public` is the canonical leartech precedent (matches `mikelear/preview-shift-left`, `mikelear/qa-architecture`)
- The repo lands with **no default branch and no initial commit** — first push creates main

**Lessons**:

1. **No `--clone` flag in this invocation** — we wanted to control the clone source (template, not empty repo). Don't combine `gh repo create` with `--clone` if you're going to populate from a template.
2. **OWNERS file appears auto-merged** — based on prior `qa-architecture` experience (`aae7838 chore: add OWNERS file`), some external automation adds an OWNERS file to new repos under `mikelear/`. The agent should expect this and not be surprised when post-creation `git pull` brings down a commit it didn't make.
3. **Branch protection** — defaults to none. Production repos eventually need branch-protection rules; new repos don't have them automatically.

---

## 2. Template clone-and-rename — **golden template's README is INCOMPLETE**

**Critical finding 2026-05-05**: the golden Go template's documented clone-and-rename procedure is **incomplete in two ways that break first-PR pipelines**. The automated-agent's bootstrap runbook must include both fixes; following the golden template's README literally produces a broken first-PR.

### Gap 1: chart directory not renamed

The README's `find ... -exec sed` only replaces file *contents*, not directory names. So `charts/leartech-go-service-template/` stays as-is even after rename.

The `preview/helmfile.yaml.gotmpl` resolves the chart path as `../charts/$APP_NAME` where `$APP_NAME = <new-service-name>`. So Helm tries to install `../charts/leartech-qa-canary/` which doesn't exist, and `gcp/pr` (or `az/pr`) fails with:

```
Error: path "../charts/leartech-qa-canary" not found
```

**Fix**: in addition to the README's procedure, also:

```bash
git mv charts/leartech-go-service-template charts/<new-service-name>
sed -i.bak "s|name: leartech-go-service-template|name: <new-service-name>|" charts/<new-service-name>/Chart.yaml
rm -f charts/<new-service-name>/Chart.yaml.bak
```

### Gap 2: bare-name (non-fully-qualified) substitution missing

The README's sed pattern is:

```bash
OLD=github.com/mikelear/leartech-go-service-template
NEW=github.com/mikelear/<new-name>
```

This only replaces the FULL module path. Bare references to `leartech-go-service-template` (without the `github.com/mikelear/` prefix) survive — and there are many of them:

- `charts/<service>/values.yaml` — `app.kubernetes.io/name: leartech-go-service-template` (becomes the K8s resource label; downstream selectors break)
- `charts/<service>/values.schema.json` — same pattern
- `cmd/server/main.go` — package comment + `Msg("starting leartech-go-service-template")` log line
- `internal/handlers/example.go`, `example_test.go` — example struct names
- `docs/swagger.yaml`, `docs/swagger.json`, `docs/docs.go` — `example: leartech-go-service-template` field
- `.golangci.yml` — depguard rule referencing the import path
- `README.md` + `CLAUDE.md` — docs

**Fix**: extend the sed to substitute the bare name across the same file extensions:

```bash
OLD_BARE=leartech-go-service-template
NEW_BARE=<new-service-name>
find . -type f \( -name '*.go' -o -name '*.yaml' -o -name '*.yml' -o -name '*.json' -o -name '*.md' -o -name 'go.mod' -o -name 'Makefile' -o -name 'Dockerfile' \) \
  -not -path './.git/*' \
  -not -path './.angular/*' \
  -exec sed -i.bak "s|$OLD_BARE|$NEW_BARE|g" {} \; \
  -exec rm -f {}.bak \;
```

This is run AFTER the module-path sed, before `git init`.

### Lessons

1. **Golden template README is currently incomplete**. The automated-agent's runbook must override it with the corrected procedure (both gaps above). Worth raising a PR against the template's README to fix this for the next session — but until that PR lands, the agent must not trust the README literally.
2. **Verification command** the agent should run BEFORE pushing the initial commit:

   ```bash
   # No bare references to the old name (excluding intentional source-attribution mentions if any):
   grep -rln "leartech-go-service-template" . 2>/dev/null | grep -v "\.git/\|\.angular/" | head
   # Empty output = clean. Non-empty = gap; investigate.
   ```

3. **`go build ./... && go test ./...`** after rename catches Go-side import path issues. Add to the agent's pre-push check.
4. **Helm chart sanity** — `helm template ./charts/<name>` will catch chart-dir-not-found errors locally before pushing. Add to pre-push check.
5. **The first-PR-of-a-new-repo failure modes are predictable** — if first-PR fails on `pr` step (not lint or test), the cause is almost certainly one of these two bootstrap gaps. The agent should retry with the comprehensive fixes before treating it as a real failure.

## 2.5 Template clone-and-rename — original procedure (now augmented)

**Pattern (from golden template README)**:

```bash
git clone --depth=1 https://github.com/mikelear/leartech-go-service-template.git <new-name>
cd <new-name>
rm -rf .git && git init -b main
OLD=github.com/mikelear/leartech-go-service-template
NEW=github.com/mikelear/<new-name>
find . -type f \( -name '*.go' -o -name 'go.mod' -o -name 'Makefile' -o -name '*.yaml' -o -name '*.md' \) \
  -exec sed -i.bak "s|${OLD}|${NEW}|g" {} \; -exec rm -f {}.bak \;
git remote add origin https://github.com/mikelear/<new-name>.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

**What worked**:

- The find/sed pattern correctly substitutes the module path across `*.go`, `go.mod`, `Makefile`, `*.yaml`, `*.md`. No JSON or other formats were affected (none needed renaming).
- The `.bak` file cleanup via second `-exec` is needed on macOS (BSD sed creates them).
- `git init -b main` creates main from the start — no `git branch -m master main` rename needed on modern git.

**Lessons**:

1. **macOS sed is BSD, not GNU** — needs `-i.bak` followed by `.bak` cleanup. GNU sed (Linux containers) just needs `-i ''`. Agent should use the BSD-compatible pattern (works on both) or detect the platform.
2. **`go mod tidy` should run after rename** — to verify the module path is consistent and no orphan imports remain. Output should be clean (no errors).
3. **`go build ./...` after tidy** — silent success = clean build = rename completed correctly. If errors appear, they name the file with the unfixed module path; sed missed an extension.
4. **Don't push then rename** — `git init` + sed + `git add .` + `git commit` + `git push -u` is the right order. Pushing the unrenamed clone first creates a useless first commit.
5. **Conventional-commit message for initial commit** — match leartech precedent. `Initial commit — <repo-name>` plus a short description body and the `Co-Authored-By` trailer.

---

## 3. Sandbox / config-only repo creation

A GitOps sandbox or config-only repo (no service code) is **much smaller** than a template-cloned service repo. Don't use the template clone-and-rename pattern; just init from scratch.

**Pattern**:

```bash
gh repo create mikelear/<repo> --public --description "..."
mkdir -p ~/leartech/<repo>/{<dir>,...}
cd ~/leartech/<repo>
git init -q -b main
git remote add origin https://github.com/mikelear/<repo>.git
# write README.md, OWNERS, content files
yq eval '.' <yaml-file> >/dev/null && echo OK     # validate every YAML before committing
git add .
git commit -m "Initial commit"
git push -u origin main
```

**Lessons**:

1. **Validate every YAML before committing** — `yq eval '. ' <file> >/dev/null`. The `lint` presubmit on pipeline-catalog (added 2026-05-04) catches this for catalog repos but new repos don't have it yet. Pre-commit local validation is the cheap insurance.
2. **OWNERS file every time** — leartech precedent. Required to gate merges. Single approvers list works for a one-person test-fixture repo.
3. **README explains the repo's role + companion repos** — paste a "Companion repos" section near the top so a session opening this repo cold knows the context.
4. **Schema-version YAML files from the start** — even the spike's hand-rolled YAML carries `schema_version: v1`. Forward-compat for when CUE schemas land.

---

## 4. Source-config registration (jx-build-cluster-gsm + jx-build-cluster-akv)

Required step before pipelines fire on a new repo. Without this, `git push` to a new repo creates commits but Lighthouse never sees them — there's no SourceRepository CRD, no GitHub webhook, no Tekton trigger.

**Pattern**:

```bash
# For each cluster repo (gsm = GCP, akv = Azure):
cd ~/leartech/jx-build-cluster-<cluster>
# 1. Ensure on main + clean working tree
git checkout main
git pull --ff-only
# 2. Edit .jx/gitops/source-config.yaml — append entries under mikelear group
#    Each entry: { name: <repo-name>, description: <one-liner> }
# 3. Validate
yq eval '.' .jx/gitops/source-config.yaml >/dev/null
# 4. Commit + push
git add .jx/gitops/source-config.yaml
git commit -m "chore(source-config): register <repos>"
git push
```

**Lessons**:

1. **Both clusters must be updated** — leartech runs GCP + Azure (`jx-build-cluster-gsm` + `jx-build-cluster-akv`). Each has its own source-config.yaml. Pipelines on either cluster require registration on that cluster. Forgetting one cluster = silent half-coverage.
2. **The cluster repos may be on feature branches with WIP** — checked out `~/leartech/jx-build-cluster-akv/` and found it on `feat/source-config-sync-and-monorepos` with unrelated in-progress work. Always check `git branch --show-current` before editing; switch to `main` and pull before applying spike changes. Don't piggyback on someone's WIP branch.
3. **Cluster repos churn frequently** — the gsm repo had ~7 unrelated commits between when I first cloned and when I tried to push. `git pull --rebase` is the right pattern; expect conflicts on auto-generated files (jx-verify, replicasets, etc.) and resolve with `git checkout HEAD -- <files>` to accept what's on the branch.
4. **Append at the end of the mikelear group** — multiple entries can be added in a single commit. Order doesn't matter functionally, but appending preserves git-blame for existing entries.
5. **Description matters** — surfaces in the architect-ui dashboards + cluster docs. Match the tone of existing entries (one-liner explaining purpose). For test fixtures, explicitly say "Not for production use" so it's clear they're sandbox.
6. **Single commit with all repos** — register all the repos in one PR/commit if they ship together. Easier to revert; clearer audit trail.
7. **What happens after push (verified 2026-05-05 spike)**: git-operator on each cluster watches the source-config.yaml. On change, it reconciles by creating `SourceRepository` CRDs + installing GitHub webhooks on each new repo. **Reconcile time-to-CRD-and-webhook: <5 minutes on both clusters in this spike** (gsm + akv). Verification:

   ```bash
   # CRD exists
   kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get sourcerepository -n jx | grep <repo-name>
   kubectl --context=modern-burro get sourcerepository -n jx | grep <repo-name>

   # Webhooks installed (both clusters)
   gh api repos/mikelear/<repo-name>/hooks --jq '.[] | "\(.config.url) | \(.active)"'
   # Expect: hook-jx.jx.leartech.com active=true + hook-jx.az.leartech.com active=true
   ```

8. **First PR fires checks on both clusters** (verified 2026-05-05 PR #1 on canary): all 9 presubmits configured in canary's triggers.yaml fired on both `gcp/` and `az/` cluster prefixes. Lighthouse Merge Status went from "Not mergeable" (before checks completed) to "In merge pool" (when checks were green-or-pending). Total checks visible at peak: ~18 (9 per cluster).

9. **`end2end-ui` vs `end2end` distinction**: the **Go service template** has `end2end` (Go-style smoke). The **Angular service template** has `end2end-ui` (Playwright). When wiring a result-store extension or gate quill, **read the consumer repo's actual triggers.yaml** rather than assuming a name from the brief. The session-0-brief.md said `end2end-ui` (matching prior auth-ui examples) but the canary-from-Go-template has `end2end`. Both write to GCS the same way, but the test pack name in `required-tests/<service>.yaml` must match.

---

## 5. (Pending — first PR pipeline)

To be captured: which checks fire on first PR; common first-run failures; how to recover from cluster-environmental issues vs code issues.

---

## 6. (Pending — first release pipeline)

To be captured: postsubmit triggers, image build, helmfile promotion, source-config implications.

---

## 7. (Pending — Lighthouse trigger registration timing)

To be captured: how long after source-config push does the trigger become live; how to verify; what manual reconcile (if any) is needed.

---

## 5. (Pending — first PR pipeline)

To be captured: which checks fire on first PR; common first-run failures; how to recover from cluster-environmental issues vs code issues.

---

## 6. (Pending — first release pipeline)

To be captured: postsubmit triggers, image build, helmfile promotion, source-config implications.

---

## 7. (Pending — Lighthouse trigger registration timing)

To be captured: how long after source-config push does the trigger become live; how to verify; what manual reconcile (if any) is needed.

---

## At-end-of-spike consolidation

When Session 0 is complete:

1. Re-organize this file into a clean runbook (numbered procedure + checklist + gotchas).
2. Move to the appropriate destination (hub shared-rule vs automated-agent procedure).
3. Update `~/leartech/qa-architecture/sessions.md` live-status to reference the runbook.
4. Add a memory entry pointing future sessions at it.

The runbook should answer: "I'm an automated agent; I need to bootstrap a new repo from golden template through PR + release-to-staging without human intervention. What sequence of steps do I execute, and what do I check at each step to know it succeeded?"
