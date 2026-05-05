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

### Gap 2 + 7: bare-name AND CamelCase substitution missing

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

**Gap #7 (added during Session 2.4 bootstrap of `tempo-to-har`, 2026-05-05)**: the template also has CamelCase variants — `GoServiceTemplate` in `SwaggerServiceName` constants, type names, etc. — that don't match the kebab-case substitution above. Add a third sed pass:

```bash
OLD_CAMEL=GoServiceTemplate
NEW_CAMEL=<NewServiceCamel>   # e.g. TempoToHar
find . -type f \( -name '*.go' -o -name '*.yaml' -o -name '*.yml' -o -name '*.json' -o -name '*.md' \) \
  -not -path './.git/*' \
  -exec sed -i.bak "s|$OLD_CAMEL|$NEW_CAMEL|g" {} \; \
  -exec rm -f {}.bak \;
```

Verification (after all three sed passes):
```bash
grep -rln -E "leartech-go-service-template|GoServiceTemplate" . 2>/dev/null | grep -v "\.git/" | head
# Empty output = clean.
```

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

## 5. First PR pipeline (captured 2026-05-05)

Verified PR #1 on canary with the bootstrap fixes from item #2.

**What runs**: 9 presubmits configured in `triggers.yaml` × 2 clusters = 18 cluster-tagged checks. All passed on the fixed SHA.

**Typical durations** (cold first run):
- `lint`, `govulncheck`, `test`: 2-4 min each
- `image-scan`, `security-scan`, `ai-review`: 3-5 min each  
- `pr` (build + scan + push + preview deploy): 8-15 min
- `dynamic-scan` (preview-env-required): 8-12 min
- `end2end` (preview-env-required): 5-10 min
- Total wall-clock from PR open to all-green: ~15-20 min on first run

**Lessons**:

1. **Don't rely on `gh pr checks` until ~60-90s after PR open** — Lighthouse needs to render PipelineRuns; checks appear progressively. Polling earlier shows only `Lighthouse Merge Status pending`.
2. **Both clusters fire all checks in parallel** — wide fan-out is normal; concurrent capacity is sufficient.
3. **`Lighthouse Merge Status: In merge pool`** is the signal Tide is ready to auto-merge. **`Not mergeable. Jobs X have not succeeded.`** lists which checks are still pending or failing.
4. **`approved` label is required for auto-merge** — added by either the Lighthouse plugin (when OWNERS approve) or the auto-promotion-bot for env-promotion PRs.
5. **Failed `pr` step almost always means bootstrap gap** (item #2). Failed `lint`/`test`/`govulncheck` typically means real code issues. The differential helps the agent decide retry-with-fix vs surface to a human.

---

## 6. First release pipeline (captured 2026-05-05)

Verified leartech-qa-canary PR #1 merge to main:

- **Postsubmit fired immediately** on main merge (within seconds)
- **Both clusters' release pipelines ran** (`leartech-qa-canary-main-release-*`)
- **Total release pipeline time: ~2-3 min** per cluster (cosign-signed image build + push + jx promote)
- **Auto-opened promotion PRs**: `mikelear/jx-build-cluster-gsm` #226 + `mikelear/jx-build-cluster-akv` #116 within ~3 minutes of merge
- **PR title format**: `chore: promote leartech-qa-canary to version 0.0.1` — predictable, parseable
- **PR labels**: `size/XS`, `env/staging`, `dependency/releases/<service>` — used for routing/auto-merge

**GCP gsm PR #226 auto-merged within minutes** (Lighthouse `env/staging+updatebot` auto-merge rule). Helmfile updated → bootjob applied → deployment created in jx-staging.

**Azure akv PR #116 stayed open** — possibly different auto-merge rules on Azure. Agent should expect manual merge may be needed on the akv side until that's diagnosed.

---

## 7. First staging deployment — bootstrap gap #3 + #4 (captured 2026-05-05)

helmfile applied successfully on GCP, but **pods failed to start** with:

```
secret "leartech-qa-canary-db" not found
```

### Bootstrap gap #3: golden template defaults to Postgres-required

Chart's `values.yaml` has `database.enabled: true` + `migrations.enabled: true` by default. Chart expects an `ExternalSecret` to sync DATABASE_URL from cluster secret backend (GSM/AKV). For a fresh service with no backend entry, this fails at deployment with the missing-secret error.

**Fix for canary** (DB-less services):

```yaml
# charts/<service>/values.yaml
database:
  enabled: false
migrations:
  enabled: false
```

**Lessons**:

1. **Decide DB-or-not at clone time**, before opening the first PR. Add to runbook:
   - If service genuinely needs Postgres → provision the ExternalSecret backend entry FIRST (or accept broken first deploy and fix-forward)
   - If DB-less → set `database.enabled: false` + `migrations.enabled: false` in `values.yaml` BEFORE the initial commit
2. **Cluster's secret-store entry must exist** before deployment can run with `database.enabled: true`. Don't assume cluster will provision on demand.
3. **Image-signature verification (Kyverno) failures are non-fatal at runtime** but show as PolicyViolation events. kubelet still pulls the image OK, so pod failures showing both Kyverno warnings AND CreateContainerConfigError are caused by the latter, not the former. Don't conflate.

### Bootstrap gap #4: stale broken promotion PRs

When deployment fails with the bad version, subsequent promotion PRs accumulate. The akv-side PR for v0.0.1 stayed open after GCP merged its broken v0.0.1. **Close stale broken promotion PRs explicitly** when fixing-forward — don't let them merge with the broken version. Agent's runbook should detect this state by checking deployment health on the cluster after promotion-PR merge; if pods unhealthy, close any stale promotion PRs that would re-deploy the broken version.

---

## 8. Lighthouse trigger registration timing (captured 2026-05-05)

Verified across all three new repos: **<5 minutes** from source-config push to:
- SourceRepository CRD created on both clusters
- GitHub webhooks installed and active on both clusters
- First push/PR fires Lighthouse triggers correctly

Reconcile time depends on git-operator polling frequency; in practice fast enough that "wait 5 minutes" is a safe assumption. **No manual reconcile needed**.

---

## Pattern: debug pods over local port-forwarding

**Captured 2026-05-05 during Session 2.4** — user preference, validated as a reusable pattern.

**Rule**: when a service needs to talk to a cluster-internal resource (Tempo, Postgres, Redis, in-cluster mTLS APIs), prefer **debug Pods** running the actual binary in-cluster over port-forwarding + local invocation.

**Why**:

1. **Reuses cluster auth + networking** — no `kubectl port-forward` choreography per debug session
2. **Same code path as production** — fewer "works locally, fails in pod" surprises
3. **Reusable** — debug Pods become `make debug-*` targets that grow into a debug-script library over time
4. **Multi-cluster trivial** — same `kubectl --context=<cluster>` invocation pattern for gcp + az
5. **No cleanup** — `kubectl run --rm -i` deletes itself when done

**Pattern**:

Each Go service ships `debug/` Job manifests + a `Makefile` block:

```makefile
# Debug recipes — run service binary in-cluster against real dependencies.
# Each is a one-shot Pod that exits when the operation completes.

.PHONY: debug-synth-canary
debug-synth-canary:                      ## Synth HAR for canary's last hour, dry-run
	kubectl --context=$(CLUSTER) run tempo-to-har-debug-$$$$ \
		--image=us-central1-docker.pkg.dev/product-first/oci/tempo-to-har:$(TAG) \
		--rm -i --restart=Never \
		--env="SERVICE_NAME=leartech-qa-canary" \
		--env="WINDOW_MINUTES=60" \
		--env="CLUSTER_TAG=$(CLUSTER)" \
		-- synth --dry-run

.PHONY: debug-synth-now
debug-synth-now:                         ## Synth HAR for $(SERVICE) (real upload)
	kubectl --context=$(CLUSTER) run tempo-to-har-debug-$$$$ \
		--image=us-central1-docker.pkg.dev/product-first/oci/tempo-to-har:$(TAG) \
		--rm -i --restart=Never \
		--env="SERVICE_NAME=$(SERVICE)" \
		-- synth
```

**Invocation**:

```bash
make debug-synth-canary CLUSTER=gke_product-first_us-east1-b_tf-jx-usable-bird TAG=0.0.1
make debug-synth-now SERVICE=leartech-broker-ui CLUSTER=modern-burro TAG=0.0.1
```

**Lessons**:

1. **`debug/` directory** alongside `cmd/` for service-specific Pod manifests if a debug operation needs more than `kubectl run` env-vars (e.g. mounted secrets, init-containers).
2. **`make debug-*` targets** discoverable via `make help`. Becomes the index of "useful one-off cluster operations" without growing into ad-hoc shell scripts.
3. **CLUSTER + TAG as Make vars** — explicit, no hidden defaults. Force the user to think about which cluster + which version they're hitting.
4. **`--rm -i --restart=Never`** is the magic incantation: Pod exits, gets deleted, no leftover state.
5. **Service account must allow the operations** — debug Pods inherit the namespace's default SA. If the operation needs anything beyond default RBAC, the Pod manifest in `debug/` becomes mandatory (the `kubectl run` shortcut won't suffice).

**For tempo-to-har specifically**: this means we DON'T port-forward Tempo from the cluster to localhost. We run `synth` in a Pod alongside Tempo, no port-forwarding.

---

## Gap #5: openapi-generation task on non-API services (captured Session 0c, 2026-05-05)

`leartech-gate` is a **CLI** (Tekton step), not an HTTP service — it has no `/openapi.json`, no swagger. The golden release pipeline includes `openapi-generation` which expects to find a swagger spec; on the gate's first release that step failed and aborted the whole pipeline before image push.

**Fix**: remove the `openapi-generation` task entirely from `.lighthouse/jenkins-x/release.yaml` for repos where the artefact is a binary executed as a Tekton step, not a long-running HTTP service.

**Heuristic for the agent's runbook**: at bootstrap time, ask "does this service expose an HTTP API to other services?" If no → strip `openapi-generation` from release pipeline. The golden template's release.yaml is HTTP-service-shaped; CLI repos need a deviation.

---

## Gap #6: multi-binary services need multi-`go build` Dockerfile (captured Session 0c, 2026-05-05)

The golden Dockerfile builds one binary (`./cmd/server`). Two services in this spike turned out to have a second binary as the actual workload:

- `leartech-gate` ships `/server` (template residue, unused) AND `/gate-cli` (the actual Tekton-invoked CLI)
- `tempo-to-har` ships `/server` (template residue, unused) AND `/synth` (the actual CronJob-invoked CLI)

If the second binary isn't built + COPY'd, the pipeline image is missing the workload — Tekton step / CronJob fails with "no such file". 

**Fix**: extend Dockerfile build stage:

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/<binary> ./cmd/<binary>
```

…and add the matching `COPY --from=build /out/<binary> /<binary>` to the runtime stage.

**For the agent's runbook**: any time you create a `cmd/<other>` directory beyond `cmd/server`, the Dockerfile must grow a parallel build + COPY pair. Verification: `docker run --rm <image> ls /` after build should show every binary you expect to ship.

---

## Discovered 2026-05-05: Tempo isn't installed (Session 2.4 validation blocker)

`tempo-to-har` was built on the implicit assumption that Tempo is already deployed in `jx-observability`. It isn't. Only Prometheus exists there. The synth binary runs (debug Pod confirmed image pull + binary execution), but there's no trace backend to query.

**Implication**: a precursor session is needed before tempo-to-har can produce real HARs from real traces. Captured as Session 2.4-pre in `sessions.md`. Until that lands, tempo-to-har is shelf-ready but not validatable.

**Lesson for the agent's runbook**: when scaffolding a service that consumes data from a backing service (Tempo, Mongo, Postgres, anything not-stdlib), **add an explicit pre-bootstrap check** that the backing service exists in-cluster. The check is one `kubectl get svc -A | grep <expected>` and saves a full session of "looks done, isn't validatable". For tempo-to-har the check would have been: `kubectl get svc -A | grep -i tempo`.

---

## Forward-looking signal source: Tekton CloudEvents

Captured 2026-05-05. The webcoder workstream is piloting Tekton CloudEvents (PipelineRun lifecycle pushed as CloudEvents). Worth considering for the QA architecture's Notifier framework as an additional signal source — see `open-questions.md` Q-CE1 for the assessment.

Key point for the agent's runbook: CloudEvents would be a **push** signal for "pipeline finished", complementing the **pull** signals (gh pr checks polling) the runbook currently uses. Not a replacement — both have their place. CloudEvents is faster + more reliable; polling is universally available even when CloudEvents infra is down.

Don't bake assumptions about CloudEvents availability into the runbook yet — pilot first via webcoder, evaluate, add to QA architecture later if useful.

---

## At-end-of-spike consolidation

When Session 0 is complete:

1. Re-organize this file into a clean runbook (numbered procedure + checklist + gotchas).
2. Move to the appropriate destination (hub shared-rule vs automated-agent procedure).
3. Update `~/leartech/qa-architecture/sessions.md` live-status to reference the runbook.
4. Add a memory entry pointing future sessions at it.

The runbook should answer: "I'm an automated agent; I need to bootstrap a new repo from golden template through PR + release-to-staging without human intervention. What sequence of steps do I execute, and what do I check at each step to know it succeeded?"
