# Session: `leartech-qa-crds` — bootstrap the QA CRD repo

The schema-only repo for the QA stack's Custom Resource Definitions. Mirrors `leartech-soc-crds` exactly — small Helm chart, just CRD YAML, no Go binary. Lands cluster-side first so consumers (`leartech-gate`, future `leartech-arrivals-observer`, possible future UI) all reference the same validated schemas.

After this lands:
- Three CRDs exist on both GCP + AZ clusters in `qa.leartech.com/v1alpha1`
- The arrivals-observer session can start (it creates `Arrival` CRs)
- Future improvement: migrate `leartech-gate`'s YAML reads from `qa-management/required-tests/` to `kubectl get TestRequirement` (proves the schema cluster-side, queryable via kubectl)

**Time budget**: 2-3 hours.

---

## Why CRDs, why now, why a separate repo

**Why CRDs (vs YAML files in qa-management)?**
- apiserver validates the schema → operators can't write malformed configs
- queryable via kubectl: `kubectl get testrequirements -A` shows what gates apply where
- watchable: arrivals-observer's controller reconciles on Arrival changes natively
- versionable: `v1alpha1` → `v1` migration path with conversion webhooks built into the Kubernetes machinery

**Why now?**
- The arrivals-observer (Phase 2.7) needs to *create* `Arrival` CRs as it observes ReplicaSet events. Without CRDs landed first, it can't even start.
- The gate currently reads YAML from `qa-management/required-tests/<service>.yaml` and `gate-metadata/quills.yaml`. Once CRDs exist, gate can EITHER keep reading YAML (Phase 1) OR migrate to kubectl-based reads (Phase 2 improvement). Either way, having the CRDs available unblocks the option.

**Why a separate repo (not bundled into gate or arrivals-observer)?**
- CRDs need to exist cluster-wide BEFORE any chart applies a CR of that kind. Helm hook ordering inside one chart is brittle.
- Multiple consumers reference the same schema: gate (read), arrivals-observer (read+write), future UI (read). One source of truth.
- `leartech-soc-crds` already proved the pattern works: tiny chart, deploys early, consumers come later.

---

## Three CRDs to define

### `TestRequirement` (cluster-scoped)

Per-service declaration of which tests must have run successfully before the gate allows promotion. Currently lives in `qa-management/required-tests/<service>.yaml`; CRD makes it kubectl-queryable.

```yaml
apiVersion: qa.leartech.com/v1alpha1
kind: TestRequirement
metadata:
  name: leartech-auth-service
spec:
  service: leartech-auth-service
  requiredTests:
    - name: smoke
      type: end2end-ui
    - name: legal-offering
      type: end2end-ui
    - name: oauth-flow
      type: end2end
  optional:
    - name: load-budget
      type: load
status:
  observedGeneration: 1
  lastSyncedFrom: qa-management@<sha>   # if there's a sync controller pulling from YAML
```

**Scope decision**: cluster-scoped. Service identity is cluster-wide; the same `leartech-auth-service` exists in `jx-staging`, `jx-preproduction`, etc.

### `Quill` (cluster-scoped)

Gate quill config. Currently lives in `qa-management/gate-metadata/quills.yaml`. The gate's gate-cli iterates these and applies each quill's `impl` against helmfile changes.

```yaml
apiVersion: qa.leartech.com/v1alpha1
kind: Quill
metadata:
  name: testing
spec:
  name: testing
  impl: result-store-lookup     # one of: result-store-lookup, helmfile-diff, post-deploy-tests
  blockRelease: true            # vs alert-only
  configRef:                    # optional — points at a ConfigMap with impl-specific config
    name: testing-quill-config
    namespace: jx
```

### `Arrival` (namespaced — typically jx-staging)

Created by the arrivals-observer when it sees a new ReplicaSet rolled out. Records the deploy event + tracks test results as the observer dispatches & polls. Read by the post-deploy-tests quill at promotion-PR time.

```yaml
apiVersion: qa.leartech.com/v1alpha1
kind: Arrival
metadata:
  name: leartech-auth-service-v0-1-32-jx-staging
  namespace: jx-staging
  labels:
    qa.leartech.com/service: leartech-auth-service
    qa.leartech.com/version: "v0.1.32"
spec:
  service: leartech-auth-service
  version: v0.1.32
  replicaSet: leartech-auth-service-7c8d9b4f6
  deployedAt: "2026-05-07T14:23:00Z"
  suspectedPR: 234   # GitHub PR number, populated when known
status:
  phase: Testing       # Pending | Testing | Passed | Failed | Timeout
  tests:
    - name: smoke
      type: end2end-ui
      status: Passed
      startedAt: "2026-05-07T14:23:30Z"
      completedAt: "2026-05-07T14:25:14Z"
      retryCount: 0
      jobName: arrivals-test-leartech-auth-service-smoke-x9k4n
    - name: legal-offering
      type: end2end-ui
      status: Failed
      startedAt: "2026-05-07T14:23:30Z"
      completedAt: "2026-05-07T14:25:48Z"
      retryCount: 1
  finalizedAt: ""      # set when phase is terminal
  newlyFailed:         # populated by observer's post-finalize diff
    - legal-offering
```

**Scope decision**: namespaced. An Arrival in `jx-staging` is distinct from the same service+version arriving in `jx-preproduction`. Same service can arrive multiple times to multiple namespaces; namespace scoping prevents collision.

**Naming convention**: `<service>-<version-with-dots-as-dashes>-<namespace>`. Idempotent; observer overwrites/updates if it sees the same arrival again (e.g., after a rolling restart).

---

## Pre-flight checklist

- [ ] `gh auth status` — `mikelear` authenticated
- [ ] Both cluster contexts available: `gke_product-first_us-east1-b_tf-jx-usable-bird` + `modern-burro`
- [ ] `~/leartech/leartech-soc-crds/` cloned locally as the reference (we mirror its shape)
- [ ] Source-config write access on both `jx-build-cluster-{gsm,akv}` (we'll register the new repo there)

---

## Deliverables

### Step 1 — bootstrap the repo

Bootstrap by **mirroring `leartech-soc-crds`** rather than the golden Go template (we don't need cmd/server, internal/, etc. — just chart + CRD yaml). Manual creation:

```sh
mkdir -p ~/leartech/leartech-qa-crds/{templates,charts/qa-crds/templates}
cd ~/leartech/leartech-qa-crds
gh repo create mikelear/leartech-qa-crds --public --description "QA Platform CRDs — TestRequirement, Quill, Arrival custom resource definitions"
```

Files to create:
- `Chart.yaml` (top-level — small Helm chart)
- `OWNERS` (mikelear)
- `README.md` (short — purpose, install path)
- `templates/_helpers.tpl` (standard labels)
- `templates/testrequirement-crd.yaml`
- `templates/quill-crd.yaml`
- `templates/arrival-crd.yaml`
- `.lighthouse/jenkins-x/triggers.yaml` (so PRs run lint + helm-template)
- `.lighthouse/jenkins-x/release.yaml` (publishes the chart on tag)

**No Go code, no Dockerfile, no Makefile (other than maybe a tiny one for `helm template` + `helm lint`).**

Reference impl: `~/leartech/leartech-soc-crds/` — same Chart.yaml layout, same `app.kubernetes.io/part-of: leartech-qa` label convention, same release pipeline.

### Step 2 — write the CRD YAML

Three files, each ~80-150 lines. Schemas above are starting points. Specific things to decide:

- **`Arrival.status` printer columns**: `kubectl get arrival` should show service, version, phase, tests-passed/failed counts. Use `additionalPrinterColumns` per CRD version.
- **Default values**: `Arrival.status.phase` defaults to `Pending` via `default: Pending` in the schema (apiserver enforces).
- **Validation**: enums on `phase`, `impl`, `type` fields. Pattern validation on `version` (must match `v\d+\.\d+\.\d+` or similar).
- **Subresources**: `status` should be a subresource on Arrival so the observer can update status without permission to mutate spec.

Mirror the validation rigour in `leartech-soc-crds/templates/securitypolicy-crd.yaml` — that's a good reference.

### Step 3 — release pipeline

Mirror `leartech-soc-crds`'s release.yaml. Publishes to `us-central1-docker.pkg.dev/product-first/oci-charts/qa-crds:<version>` on tag (same OCI chart registry as other leartech charts).

### Step 4 — register in source-config on both clusters

Add to **both** `jx-build-cluster-{gsm,akv}/.jx/gitops/source-config.yaml`:

```yaml
- name: leartech-qa-crds
  description: "QA Platform CRDs — TestRequirement, Quill, Arrival custom resource definitions"
```

Two PRs (one per cluster repo). Triggers Lighthouse webhook installation on the new repo automatically (verified pattern from earlier QA-stack repos).

### Step 5 — install on both clusters

Add a release to a new helmfile `helmfiles/qa-crds/helmfile.yaml` on each cluster (mirrors `cnpg-system/helmfile.yaml` shape):

```yaml
environments:
  default:
    values:
    - jx-values.yaml
---
namespace: leartech-qa
repositories:
- name: dev
  url: us-central1-docker.pkg.dev/product-first/oci-charts
  oci: true
releases:
- chart: dev/qa-crds
  version: 0.1.0
  name: qa-crds
  values:
  - jx-values.yaml
```

Add `- path: helmfiles/qa-crds/helmfile.yaml` to each cluster's top-level `helmfile.yaml` (after `cnpg-system`, before any consumer that might create CRs of these kinds).

**Where to install** — use a new namespace `leartech-qa` (cluster-scoped CRDs don't really belong to a ns, but Helm needs *some* release namespace; same pattern as `cnpg-system`).

### Step 6 — validate

```sh
# On both clusters, after boot reconcile:
kubectl get crd | grep qa.leartech.com
# Expect: testrequirements.qa.leartech.com, quills.qa.leartech.com, arrivals.qa.leartech.com

# Apply a sample TestRequirement (from this repo's examples/ dir):
kubectl apply -f examples/testrequirement-leartech-qa-canary.yaml
kubectl get testrequirement leartech-qa-canary -o yaml
# Should pretty-print with our additionalPrinterColumns

# Apply a sample Arrival:
kubectl apply -f examples/arrival-canary-v0.0.4.yaml
kubectl get arrival -A
```

---

## Out of scope (deferred)

- **Conversion webhooks** for v1alpha1 → v1 migration. We're starting at v1alpha1; bump to v1 only when the schemas are stable for ~1-2 months.
- **Validating webhooks** beyond the OpenAPI schema. CEL validation rules (Kubernetes 1.30+) cover most need; admission webhooks are a follow-up if/when we need cross-resource validation.
- **A controller for `Arrival.status.suspectedPR` autopopulation**. The observer (Phase 2.7) does this; CRDs alone don't.
- **Migrating gate to read TestRequirement CRs instead of YAML**. The CRD existing makes that possible; the migration is its own session (~2-3h).
- **CRD versioning + storage migration**. Standard pattern; we'll cross that bridge at v1alpha1 → v1.

---

## Likely gotchas

1. **Helm release `--namespace leartech-qa` doesn't actually scope CRDs** — they're cluster-resources. The release ns is just bookkeeping. Make sure the namespace is created via a dedicated chart resource (or by `helmfile`'s `createNamespace: true`) before the chart applies, otherwise Helm fails on "namespace not found" before installing CRDs.
2. **CRD update vs replace** — Helm v3 by default *only updates* CRDs on `helm install`, NOT on `helm upgrade`. Standard workaround: put CRDs in `crds/` directory of the chart, OR use `meta.helm.sh/hook: pre-install,pre-upgrade`. soc-crds uses templates/ — verify that approach actually re-applies on upgrade. If not, switch to `crds/` dir.
3. **`additionalPrinterColumns` on Arrival** — `kubectl get arrival` is human-friendly only with these. Easy to forget; lint via `kubectl explain` after apply.
4. **kubectl apply rejects unknown fields** — if observer's first version writes a field we forgot to declare in the schema, apply fails silently (status update doesn't show in `kubectl get -o yaml`). `x-kubernetes-preserve-unknown-fields: true` on `status` while we're at v1alpha1 is forgiving; tighten when stable.
5. **CRD scope mismatch in CR** — applying a `TestRequirement` (cluster-scoped) with a `metadata.namespace` field set is rejected. Document explicitly in the chart README + examples/.

---

## Done criteria

- [ ] `mikelear/leartech-qa-crds` repo exists with chart + 3 CRD templates + release pipeline
- [ ] Repo registered in both `jx-build-cluster-{gsm,akv}` source-configs
- [ ] First release tag `v0.1.0` published to GAR OCI charts
- [ ] `helmfiles/qa-crds/helmfile.yaml` added on both clusters; top-level `helmfile.yaml` references it
- [ ] After boot reconcile, `kubectl get crd | grep qa.leartech.com` shows 3 CRDs on both clusters
- [ ] Sample CRs apply cleanly: TestRequirement (cluster-scoped), Quill (cluster-scoped), Arrival (in jx-staging)
- [ ] `examples/` dir in repo with one valid CR per CRD for human reference
- [ ] `~/leartech/qa-architecture/sessions.md` updated with new session row + brief link
- [ ] `~/leartech/hub/shared-rules/secrets-and-tekton.md` cleanup-backlog item #2 (broken GCP `tekton-container-registry-auth`) is unrelated but worth noting if the helmfile reconcile surfaces it again

---

## After this lands

- **Next session: Phase 2.7.1** (`leartech-arrivals-observer` skeleton + K8s ReplicaSet watcher, ~4-5h). Brief lives at `session-2-7-1-arrivals-observer-bootstrap-brief.md` (to be drafted).
- **Optional follow-up**: gate migrates from YAML reads → CR reads. Tackle when arrivals-observer is producing Arrivals so the migration has real traffic to validate against.
- **Adoption signal**: `kubectl get testrequirements -A` becomes the canonical answer to "what does this service need to pass before promotion?". qa-management YAML stays as the human-editable source; a syncer keeps CRs in sync (or a Renovate-style PR-against-qa-management generates the CR change too — TBD when we tackle the gate-migration session).
