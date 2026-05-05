# Session: Platform Foundations — Tempo + CNPG Postgres

Two infra installs that unblock multiple downstream sessions. Both are dev/staging-only on the GCP cluster (Azure deferred to Phase 1 hardening); production gets the same treatment in a later session once the staging pattern is validated.

After this lands:
- `tempo-to-har` (Session 2.4) can be validated end-to-end against real spans
- New services that need a DB consume a shared CNPG `Database` instead of bundling their own Bitnami chart
- Golden Go template emits OTLP traces by default, so every new repo is tracing-ready from day 1

**Time budget**: 4-5 hours.
**Cluster scope**: GCP staging only (`gke_product-first_us-east1-b_tf-jx-usable-bird`). Azure parity is a Phase 1 hardening item, batched with the cross-cloud GAR auth fix.

---

## Why these two together

Both are "things every new service implicitly needs" that we've been working around with per-service bundling. mqube has solved this pattern already — copy what's known to work, deviate only where leartech's setup demands it.

| Problem today | Workaround | Cost as we scale |
|---|---|---|
| No tracing backend | services don't emit OTLP; tempo-to-har can't be validated | every new service ships without observability hooks; debugging cross-service flows = grep through logs |
| No shared DB | golden template bundles per-namespace Bitnami Postgres; auth-service preview pattern copied to webcoder-ui | 8+ Postgres pods on GCP, 6+ on AZ today; ~3-6 GiB RAM wasted; each new service that needs a DB doubles the operational surface |

Both have the same shape of fix: install one operator/service in `jx-observability` (Tempo) or `infrastructure` (CNPG), declare per-service consumption in the service's own helm values, and update the golden template so new repos pick up the pattern automatically.

---

## What mqube does (reference for both halves)

**Tempo** (`~/mqubeRepos/JX3_Azure_Vault_Dev_Cluster/helmfiles/jx-observability/helmfile.yaml:60-68`):
- Chart: `grafana-community/tempo:2.0.0` (single-binary, OCI from `ghcr.io/grafana-community/helm-charts`)
- OTLP HTTP 4318 + gRPC 4317 + Jaeger receivers exposed
- 30d retention, persistence enabled (local PV)
- `node: infra` selector + spot tolerations (Azure-specific, drop for GCP)
- Labelled `values.jenkins-x.io: lock` (no auto-promotion)

**Tempo in production**: NOT installed. Only Prom + Grafana + Loki + Alloy. Validates that Tempo is dev/staging-only — traces are debugging tooling, not prod-critical.

**Collector**: Alloy 1.7.0 — but currently logs-only (Loki ingest with PII redaction). OTLP ports are exposed but not routed to Tempo. Services in mqube push OTLP **straight to Tempo**, bypassing Alloy. Adopt the same pattern: services → Tempo direct. Alloy stays logs-focused for now; we can add OTel routing later if we need fan-out.

**Postgres**: mqube has no shared CNPG cluster. Each service that needs Postgres bundles it. They live with the sprawl. **We're going further than mqube here** — picking the operator-managed pattern up front because the Bitnami-per-namespace cost will accelerate as we copy the auth pattern to more services.

---

## Pre-flight checklist

- [ ] `kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get ns jx-observability infrastructure` — both exist (we already saw them)
- [ ] `kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n jx-observability get svc` — only Prometheus, no Tempo (confirms we're starting clean)
- [ ] `~/leartech/jx-build-cluster-gsm/` checked out clean — this is the GCP cluster's GitOps repo
- [ ] `gh auth status` — `mikelear` authenticated
- [ ] mqube Tempo + alloy reference files locally available at `~/mqubeRepos/JX3_Azure_Vault_Dev_Cluster/helmfiles/jx-observability/`

---

## Part 1 — Tempo (~1.5h)

### Step 1.1 — Add Tempo release to GCP cluster's jx-observability helmfile

`~/leartech/jx-build-cluster-gsm/helmfiles/jx-observability/helmfile.yaml` (or wherever leartech keeps observability config — verify path before editing). Mirror mqube's release definition:

```yaml
- chart: grafana-community/tempo
  version: 2.0.0
  name: tempo
  labels:
    values.jenkins-x.io: lock
  values:
  - configs/tempo.yaml
  - jx-values.yaml
```

`configs/tempo.yaml` (start from mqube's, drop Azure-isms):

```yaml
tempo:
  retention: 720h  # 30d. Match mqube's value but in cleaner units.
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317
    jaeger:
      protocols:
        grpc:
          endpoint: 0.0.0.0:14250
        thrift_http:
          endpoint: 0.0.0.0:14268
persistence:
  enabled: true
  size: 50Gi  # 30d retention at expected leartech volume — revisit when we have data
podAnnotations:
  prometheus.io/port: prom-metrics
  prometheus.io/scrape: "true"
# No node selectors / tolerations — leartech GCP doesn't have an infra pool yet.
```

If the OCI repo isn't already declared in the helmfile, add it under `repositories`:

```yaml
- name: grafana-community
  url: ghcr.io/grafana-community/helm-charts
  oci: true
```

### Step 1.2 — Open the helmfile change as a normal infra PR

Standard flow: branch → push → PR → wait for `verify` + (eventually) qa-gate to pass → merge → boot reconciles → Tempo running.

Validate post-reconcile:

```bash
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n jx-observability get pods,svc | grep tempo
# Expect tempo-0 Running 1/1, tempo svc with 4317+4318 ports
```

### Step 1.3 — Wire Tempo as a Grafana datasource

Inspect the existing `~/leartech/jx-build-cluster-gsm/helmfiles/jx-observability/configs/grafana.yaml` for the datasource list (or wherever leartech defines it). Add:

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo.jx-observability.svc.cluster.local:3200
        uid: tempo
        editable: false
```

Reload Grafana (helm upgrade picks it up via gitops boot).

### Step 1.4 — Validate Tempo reachable from a debug Pod

Use the debug-Pod pattern (Session 0c lesson):

```bash
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n jx-staging run tempo-probe --image=curlimages/curl --rm -i --restart=Never --command -- \
  curl -sf http://tempo.jx-observability.svc.cluster.local:3200/ready
# Expect HTTP 200, "ready" body.
```

### Step 1.5 — Add OTLP exporter to golden Go template

`~/leartech/leartech-go-service-template/cmd/server/main.go` — initialize `otel/sdk` with OTLP HTTP exporter. Default endpoint: `http://tempo.jx-observability.svc.cluster.local:4318` (configurable via `OTEL_EXPORTER_OTLP_ENDPOINT`). Always-sample for staging, ratio-sample (1%) for production via env.

```go
// Pseudocode — actual imports + provider setup goes here.
exporter, _ := otlptracehttp.New(ctx,
    otlptracehttp.WithEndpoint(getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "tempo.jx-observability:4318")),
    otlptracehttp.WithInsecure(),
)
tp := sdktrace.NewTracerProvider(
    sdktrace.WithBatcher(exporter),
    sdktrace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName(serviceName),
        attribute.String("cluster", os.Getenv("CLUSTER_TAG")),
    )),
)
otel.SetTracerProvider(tp)
```

Add `otelhttp.NewHandler(...)` wrapper to the chi router so every request gets a span automatically.

Update `CLAUDE.md` in the template with: "tracing is on by default; to opt out, set `OTEL_TRACES_DISABLED=1`."

### Step 1.6 — Re-test tempo-to-har against the canary

Once new auth-service / canary deploys carry traces:

```bash
cd ~/leartech/tempo-to-har && make debug-synth-canary TAG=0.0.1
```

Expect: real OTLP spans returned from Tempo, HARs synthesized in-memory (dry-run), exit 0. Then:

```bash
make debug-synth-now SERVICE=leartech-qa-canary CLUSTER=gke_product-first_us-east1-b_tf-jx-usable-bird TAG=0.0.1
# Real GCS upload. Verify: gsutil ls gs://test-artifacts-product-first/har/v1/tempo-synth/leartech-qa-canary/
```

This is the unblocker for Session 2.4. After Step 1.6 succeeds, mark 2.4 ✅ in `sessions.md`.

---

## Part 2 — CNPG Postgres (~2-2.5h)

### Step 2.1 — Install CNPG operator

CloudNativePG is the modern, operator-managed alternative to bundled Bitnami Postgres. It runs in `cnpg-system` namespace and watches `Cluster` + `Database` + `Pooler` CRs cluster-wide.

Install via helm release in `~/leartech/jx-build-cluster-gsm/helmfiles/cnpg-system/helmfile.yaml` (new directory):

```yaml
namespace: cnpg-system
repositories:
- name: cnpg
  url: https://cloudnative-pg.github.io/charts
releases:
- chart: cnpg/cloudnative-pg
  version: 0.22.1  # check latest before committing
  name: cnpg-operator
  labels:
    values.jenkins-x.io: lock
  values:
  - jx-values.yaml
```

Wire into the cluster's top-level helmfile-of-helmfiles so boot reconciles it.

Validate post-reconcile:

```bash
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n cnpg-system get pods
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get crds | grep cnpg
# Expect cnpg-controller-manager Running, Cluster + Database CRDs present.
```

### Step 2.2 — Create one staging Postgres `Cluster`

`~/leartech/jx-build-cluster-gsm/helmfiles/cnpg-clusters/staging.yaml` (new):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: leartech-staging
  namespace: infrastructure
spec:
  instances: 2  # 1 primary + 1 replica
  imageName: ghcr.io/cloudnative-pg/postgresql:17.2
  storage:
    size: 20Gi
    storageClass: standard-rwo
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      memory: 1Gi
  monitoring:
    enablePodMonitor: true  # Prometheus picks it up automatically
  bootstrap:
    initdb:
      database: postgres
      owner: postgres
```

Phase 2 (later session) — add `backup:` block writing WAL + base backups to GCS for PITR. Skip for staging spike.

Validate:

```bash
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n infrastructure get cluster
# Expect leartech-staging  2/2  PRIMARY  HEALTHY
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n infrastructure get svc | grep leartech-staging
# Expect leartech-staging-rw (writes), leartech-staging-ro (read-replica), leartech-staging-r (any).
```

### Step 2.3 — Provision auth-service's database via CNPG `Database` CR

`~/leartech/leartech-auth-service/charts/leartech-auth-service/templates/postgres-database.yaml` (new — gated by `.Values.postgresql.useSharedCluster`):

```yaml
{{- if .Values.postgresql.useSharedCluster }}
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: {{ .Release.Name }}-auth-service
spec:
  cluster:
    name: leartech-staging
  name: auth_service
  owner: auth_service  # CNPG creates the role automatically
---
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: {{ .Release.Name }}-hydra
spec:
  cluster:
    name: leartech-staging
  name: hydra
  owner: hydra
{{- end }}
```

Add `useSharedCluster: false` to `values.yaml` (default off). Staging helmfile overlay flips it on; previews stay on the bundled Bitnami chart untouched.

CNPG generates a Secret per role (`leartech-staging-app` style); ExternalSecret pulls the DSN from there into the auth-service Pod env.

### Step 2.4 — Migrate auth-service jx-staging to CNPG

`~/leartech/<staging-helmfile-repo>/helmfiles/staging/helmfile.yaml` — for the auth-service release, add:

```yaml
values:
- ...existing...
- inline:
    postgresql:
      useSharedCluster: true
      secretName: leartech-staging-auth_service-app  # CNPG-generated
      secretKey: uri
    storeBackend: postgres
```

Drop the bundled `auth-postgresql` from the staging deploy (preview helmfile is unchanged).

Migration sequence on cutover PR:
1. Take a `pg_dump` of the existing `auth-postgresql` StatefulSet (data in `auth_service` DB only — Hydra config can be re-bootstrapped).
2. Apply the helmfile change. CNPG `Database` CRs create empty `auth_service` + `hydra` DBs in the shared cluster.
3. `psql` restore the dump into `leartech-staging-rw`.
4. Roll auth-service Deployment so it picks up the new DSN.
5. Validate login flow: hit `https://auth-ui-jx-staging.jx.leartech.com`, log in with seeded user.
6. Once green for 24h, delete the orphaned `auth-postgresql` StatefulSet from jx-staging (separate PR — easy rollback if step 5 reveals a regression).

### Step 2.5 — Update golden Go template docs

`~/leartech/leartech-go-service-template/CLAUDE.md` — add a "Database" section:

> If your service needs a Postgres DB, **do not** add a Bitnami chart dependency. Instead, in your service's chart:
>
> 1. Add a `Database` CR template gated on `.Values.postgresql.enabled`. Owner = `<service-name>`. Cluster ref = `leartech-staging`.
> 2. Read the DSN from the CNPG-generated secret via env var.
> 3. For PR previews: helmfile `useSharedCluster: false` keeps the per-namespace Bitnami pattern. For staging+production: `useSharedCluster: true`.
>
> Don't multi-tenant your DB across services. One `Database` per service. Schema migrations live in your service repo and run as a `bootstrap.initdb.postImportApplicationSQL` snippet or a Job — not in the cluster manifest.

---

## Out of scope (deferred)

- **Azure parity for Tempo + CNPG**: Phase 1 hardening — batched with the cross-cloud GAR auth fix from Session 0c.
- **Production CNPG cluster**: install once staging cluster has been live for 2-4 weeks. Add WAL/PITR backups to GCS at that point.
- **Tempo in production**: deferred indefinitely (mqube doesn't run it in prod either; revisit if forensics need it).
- **Migrating webcoder-ui PR-7's bundled Postgres** off Bitnami: previews stay on Bitnami; only staging/prod need to migrate.
- **Loki + Alloy improvements**: leartech has neither installed today (only Prometheus). Logs are a separate session — flag in `sessions.md` but don't block this one on it.
- **OTel collector (Alloy)**: services push OTLP to Tempo direct, mirroring mqube. If we later need fan-out (Tempo + a SaaS APM), we add an OTel collector in front. Not a Phase 0 problem.

---

## Likely gotchas (anticipated from Sessions 0/0c/2.4 lessons)

1. **OCI chart auth**: if `ghcr.io/grafana-community/helm-charts` requires auth, fail-fast — fall back to the legacy `https://grafana.github.io/helm-charts` and log it as a bootstrap gap.
2. **CNPG storageClass mismatch**: GCP default is `standard-rwo` but verify with `kubectl get storageclass`. If `standard` (no `-rwo`), 2-replica clusters can't bind PVs.
3. **OTLP from a service to Tempo across namespaces**: NetworkPolicy may block egress from `jx-staging` → `jx-observability`. Verify with the debug-Pod curl from Step 1.4 *from a Pod in jx-staging*, not the default ns.
4. **CNPG `Database` CR secret naming**: format is `<cluster-name>-<role>-app`. Read CNPG release notes — name pattern has changed across operator versions.
5. **Hydra schema bootstrap**: Hydra runs `hydra migrate sql` against its DB on startup. Make sure the `hydra` role has the right grants on the `hydra` DB — CNPG Database CR sets `owner` correctly but Hydra wants CREATE on schema `public`, which Postgres 15+ removed by default.

---

## Done criteria

- [ ] `kubectl get pods -n jx-observability | grep tempo` → 1/1 Running
- [ ] Grafana shows Tempo as a datasource; can run a TraceQL query and see auth-service spans (Session 1.5 wires that)
- [ ] `make debug-synth-canary` in tempo-to-har returns real spans + synthesizes HARs (Session 2.4 unblocked)
- [ ] `kubectl get cluster -n infrastructure` → `leartech-staging  2/2  PRIMARY  HEALTHY`
- [ ] `kubectl get database -n jx-staging` → `auth_service`, `hydra` both Healthy
- [ ] auth-service jx-staging login flow green after migrating to CNPG DSN
- [ ] Old `auth-postgresql` StatefulSet in jx-staging deleted (separate PR; not blocking)
- [ ] Golden Go template's `CLAUDE.md` has the Database + OTLP-tracing sections
- [ ] `~/leartech/qa-architecture/sessions.md` marks Session 2.4 ✅ and Session 2.4-pre ✅; new "Platform Foundations" row added
- [ ] `~/leartech/qa-architecture/session-0-lessons.md` updated with any new bootstrap gaps surfaced (likely candidates: OCI chart auth, NetworkPolicy egress, Hydra schema grants)

---

## After this session

- **Session 1.x (post-spike hardening)** picks up cleanly: services emit traces by default, qa-management has a stable schema target.
- **Phase 2 (incident-response stuff)** has the foundations it needs: Tempo for forensics, Postgres for any audit-log/event-store work.
- **Tenant onboarding** later in Phase 2 inherits the same pattern — tenant repos add a `Database` CR pointing at the same shared cluster (or get their own per-tenant cluster if isolation is required).
