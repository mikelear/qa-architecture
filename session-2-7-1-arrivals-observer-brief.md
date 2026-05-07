# Session 2.7.1 — `leartech-arrivals-observer` repo + `Arrival` CRD

**Revised 2026-05-07**: dropped the separate `leartech-qa-crds` repo. Following the proven Maestro/CNPG/Hydra-Maester pattern instead — CRDs bundled directly into the consumer chart. Reference impls: `mqube-maestro-service/charts/mqube-maestro-service/config/maestro.mqube.com_eventregistrations.yaml`, plus our own `cnpg-operator` install which puts CRDs in the operator's chart templates.

**Time budget**: 4-5 hours.

---

## Outcome we're proving

```
ReplicaSet Added event in jx-staging
        ↓
arrivals-observer's K8s informer fires
        ↓
Redis lock acquired (per service+version, dedup across replicas)
        ↓
Arrival CR created in jx-staging:
  apiVersion: qa.leartech.com/v1alpha1
  kind: Arrival
  metadata: { name: <service>-<ver>-jx-staging, namespace: jx-staging }
  spec: { service, version, replicaSet, deployedAt }
  status: { phase: Pending }
        ↓
kubectl get arrivals -A
NAMESPACE    NAME                                       SERVICE              VERSION    PHASE     AGE
jx-staging   leartech-auth-service-v0.1.32-jx-staging   leartech-auth-svc    v0.1.32    Pending   3s
```

That's the Phase 2.7.1 finish line: arrivals-observer is **observing + recording**. Test dispatch + status finalization is Phase 2.7.2.

---

## Why this revision dropped the separate `*-crds` repo

Earlier brief proposed `leartech-qa-crds` mirroring `leartech-soc-crds`. But:

1. **`leartech-soc-crds` is NOT actually deployed** — it exists as a chart skeleton + source-config registration, but no helmfile entry references it on either cluster. The "proven pattern" claim was wrong; it was a planned pattern, never exercised.
2. **What IS proven on cluster**:
   - `mqube-maestro-service` bundles its `EventRegistration` CRD inside the chart at `charts/<chart>/config/maestro.mqube.com_eventregistrations.yaml`
   - Hydra Maester bundles `OAuth2Client` CRD inside the hydra-maester subchart of mqube-auth-service
   - CNPG operator (we just installed it 2026-05-05) bundles `Cluster` + `Database` + `Pooler` + 7 other CRDs inside the operator chart
3. **JX3 routes CRDs by kind regardless of source location** — they ALL end up in `config-root/customresourcedefinitions/` and get applied before namespaced resources. No structural difference between bundled and separate-repo approaches.

So the call: bundle CRDs into the consumer chart for spike velocity. Refactor to a separate repo only if 3+ services genuinely share the same schema (and we have evidence today only the observer writes `Arrival`; gate just reads via kubectl).

---

## Scope decision: only `Arrival` CRD in this session

Brief originally proposed three CRDs (`TestRequirement` + `Quill` + `Arrival`). Trimming for Phase 2.7.1:

- **`Arrival`** — needed for observer to record events. **In scope.** Bundle in observer chart.
- **`TestRequirement`** — would replace `qa-management/required-tests/<service>.yaml` reads at PR-time. The gate works fine with YAML today. Migration to CRD is optional Phase 2 polish; **not Phase 2.7.1 scope**.
- **`Quill`** — same: gate reads `qa-management/gate-metadata/quills.yaml` today. Works. Migration is optional. **Not in scope.**

If/when we tackle the YAML→CRD migration for gate config, those CRDs would be bundled into `leartech-gate`'s chart (since gate is the only reader/writer at that point).

---

## Pre-flight checklist

- [ ] CNPG `Database` + `Cluster` CRDs are HEALTHY on both clusters (verifies the bundled-CRD-in-operator-chart pattern works for us)
- [ ] Both cluster contexts available
- [ ] Source-config write access on both `jx-build-cluster-{gsm,akv}`
- [ ] `~/leartech/leartech-go-service-template/` checked out — we clone-and-rename
- [ ] All 7 bootstrap-gap fixes from `~/leartech/qa-architecture/session-0-lessons.md` available, including the CamelCase substitution (`GoServiceTemplate` → `LeartechArrivalsObserver`)

---

## Deliverables

### Step 1 — bootstrap `leartech-arrivals-observer` from golden Go template

Clone-and-rename per `session-0-lessons.md` runbook (3 sed passes: full-module-path, bare-name, CamelCase). Verification command:

```sh
grep -rln -E "leartech-go-service-template|GoServiceTemplate" . 2>/dev/null \
  | grep -v "\.git/\|\.angular/" | head
# Empty output = clean. Non-empty = bootstrap gap recurrence; fix before proceeding.
```

Apply Session 0c gaps:
- Drop `openapi-generation` from `.lighthouse/jenkins-x/release.yaml` (this is a controller, not an HTTP API service)
- Dockerfile is a single-binary build (no `cmd/<other>` here yet) — no Gap #6 needed yet

### Step 2 — register on both clusters

Add to **both** `jx-build-cluster-{gsm,akv}/.jx/gitops/source-config.yaml`:

```yaml
- name: leartech-arrivals-observer
  description: "Fat-Controller-equivalent — watches K8s ReplicaSets in jx-staging, creates Arrival CRs, dispatches post-deploy tests"
```

Two PRs (one per cluster). Lighthouse webhooks install automatically.

### Step 3 — define the `Arrival` CRD (bundled in chart)

Create `charts/leartech-arrivals-observer/config/qa.leartech.com_arrivals.yaml` (mirrors mqube-maestro layout):

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: arrivals.qa.leartech.com
  labels:
    app.kubernetes.io/part-of: leartech-qa
spec:
  group: qa.leartech.com
  scope: Namespaced
  names:
    kind: Arrival
    listKind: ArrivalList
    plural: arrivals
    singular: arrival
    shortNames: [arr]
  versions:
  - name: v1alpha1
    served: true
    storage: true
    subresources:
      status: {}
    schema:
      openAPIV3Schema:
        type: object
        required: [spec]
        properties:
          spec:
            type: object
            required: [service, version, replicaSet, deployedAt]
            properties:
              service:
                type: string
                description: Service name (matches Deployment label app.kubernetes.io/name)
              version:
                type: string
                description: Service version (matches Deployment label app.kubernetes.io/version)
              replicaSet:
                type: string
                description: Name of the ReplicaSet that triggered this arrival
              deployedAt:
                type: string
                format: date-time
              suspectedPR:
                type: integer
                description: GitHub PR number, populated when known
          status:
            type: object
            x-kubernetes-preserve-unknown-fields: true   # forgiving while at v1alpha1
            properties:
              phase:
                type: string
                enum: [Pending, Testing, Passed, Failed, Timeout]
                default: Pending
              tests:
                type: array
                items:
                  type: object
                  properties:
                    name: { type: string }
                    type: { type: string, enum: [end2end, end2end-ui, contract, unit, load] }
                    status: { type: string, enum: [Pending, Running, Passed, Failed, Timeout] }
                    startedAt: { type: string, format: date-time }
                    completedAt: { type: string, format: date-time }
                    retryCount: { type: integer, default: 0 }
                    jobName: { type: string }
              finalizedAt: { type: string, format: date-time }
              newlyFailed:
                type: array
                items: { type: string }
    additionalPrinterColumns:
    - name: Service
      type: string
      jsonPath: .spec.service
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

JX3 routes this to `config-root/customresourcedefinitions/jx-staging/leartech-arrivals-observer/config/qa.leartech.com_arrivals-crd.yaml` (or similar) on next regen, applies before namespaced resources at boot.

### Step 4 — RBAC for the observer

Service account in `jx-staging` ns needs:
- `get,list,watch` on `replicasets.apps`, `deployments.apps`, `pods` (for arrival detection)
- `get,list,create,update,patch` on `arrivals.qa.leartech.com` (the CR it manages)
- `update,patch` on `arrivals/status` (status subresource)

Add `templates/role.yaml` + `templates/rolebinding.yaml` + extend `templates/serviceaccount.yaml` from the golden template baseline.

### Step 5 — implement the K8s watcher (Go)

`internal/watcher/replicaset.go`:
- `client-go` informer scoped to `jx-staging` ns (cluster-config'd)
- ReplicaSet `Added` events trigger arrival creation
- Filter: only ReplicaSets with `app.kubernetes.io/managed-by=Helm` label (skip system / one-off)
- Redis lock keyed `arrival:<service>:<version>`, TTL 30 min, prevents duplicate processing across replicas
- Idempotent: `kubectl create -f arrival.yaml` — on conflict (already exists), update spec.deployedAt only (so a rolling restart of the same version doesn't create new Arrivals)

`cmd/server/main.go` (replacing the HTTP boilerplate with a controller setup):
- Start informer factory
- Run reconciliation loop
- `/health/live` + `/health/ready` for Kubernetes probes (kept from template)
- `/metrics` for Prometheus (kept from template)

OTLP + golden-template Makefile + go.mod toolchain pin (1.25.9) all carry over from Phase Platform Foundations work.

### Step 6 — chart wiring

In addition to Step 3's CRD, add:
- `templates/configmap.yaml` for service config (filter labels, Redis URL, etc.)
- Reuse `_helpers.tpl`, `_security.tpl`, `_resources.tpl` from golden template

Helm install order on cluster:
1. CRDs apply via JX3 routing (cluster-wide, before any namespaced resources)
2. Deployment + ServiceAccount + Role + RoleBinding apply in jx-staging

### Step 7 — register release in cluster helmfile

Add to `helmfiles/jx-staging/helmfile.yaml` on both clusters:

```yaml
- chart: dev/leartech-arrivals-observer
  version: 0.0.1
  name: leartech-arrivals-observer
  values:
  - jx-values.yaml
```

### Step 8 — validate on cluster

```sh
# Verify CRD lands
kubectl get crd arrivals.qa.leartech.com
kubectl explain arrival.spec
kubectl explain arrival.status

# Roll a known service (canary) to trigger an arrival
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird \
  -n jx-staging rollout restart deploy/leartech-qa-canary

# Watch
kubectl get arrivals -n jx-staging --watch
# Expected: a new Arrival appears within ~30s of the new ReplicaSet

# Inspect with printer columns
kubectl get arrival -n jx-staging
NAME                                       SERVICE              VERSION   PHASE     AGE
leartech-qa-canary-v0.0.4-jx-staging       leartech-qa-canary   v0.0.4    Pending   12s
```

---

## Out of scope (Phase 2.7.2-2.7.6)

- Test dispatch via K8s Job (2.7.2) — observer just records events here; doesn't kick off tests yet
- Tempo client + traffic forensics (2.7.3)
- Slack alerter (2.7.4)
- post-deploy-tests quill in leartech-gate (2.7.5)
- Retro + threshold tuning (2.7.6)

---

## Anticipated gotchas

1. **CRD apply happens in `customresourcedefinitions/` dir, not `namespaces/`.** Even though our chart's `templates/` folder is the same as for namespaced resources, JX3's regen routes by `kind`. If the regen doesn't pick up our Arrival CRD, check `make regen-phase-1` output for routing decisions.
2. **`additionalPrinterColumns`** missing → `kubectl get arrival` only shows `NAME` and `AGE`. Add columns; `kubectl explain` validates.
3. **Status subresource omitted** → controller can't update status without spec write permission. Standard CRD requirement; include `subresources.status: {}`.
4. **Informer ResyncPeriod default is 0** (no resync) — fine for spike. If we miss events, bump to 30s.
5. **Dual-cluster deployment** — observer runs in BOTH GCP + AZ jx-staging. Each instance only watches its own cluster's ReplicaSets. No cross-cluster coordination needed at Phase 2.7 — each cluster has its own Arrival ledger.
6. **Redis lock key collision** — if two clusters share a Redis (they don't today; each cluster has its own jx-redis), keys would collide on `arrival:<service>:<version>`. Phase 2 hardening: prefix with cluster ID. Skip for spike since clusters have independent Redis.
7. **`spec.deployedAt` vs `metadata.creationTimestamp`** — keep both. `metadata.creationTimestamp` is when WE recorded it; `spec.deployedAt` is when K8s thinks the ReplicaSet became active. They'll usually differ by seconds; don't conflate.

---

## Done criteria

- [ ] `mikelear/leartech-arrivals-observer` repo bootstrapped + cleanly built (zero residual `leartech-go-service-template` / `GoServiceTemplate` strings)
- [ ] Registered in both clusters' source-config; release pipeline cuts `v0.0.1`; image lands in GAR + ACR
- [ ] Helmfile entries on both clusters; boot reconcile applies cleanly
- [ ] `kubectl get crd arrivals.qa.leartech.com` returns on both clusters
- [ ] Rolling restart of `leartech-qa-canary` produces a fresh `Arrival` CR within 30s
- [ ] Status updates reflect (kubectl edit a status field, observer should NOT overwrite it — proves status subresource is honored)
- [ ] Sessions.md updated; lessons captured if any new bootstrap gap surfaced

---

## After this lands

- **2.7.2** test dispatch — observer kicks off K8s Jobs against tests defined in qa-management
- **2.7.3** Tempo client + traffic forensics — extends observer to query spans for the new arrival, diff against baseline
- Eventually: gate's post-deploy-tests quill reads `Arrival` CRs at promotion-PR time
