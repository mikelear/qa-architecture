# CNPG Staging Cutover Runbook

Cutting `leartech-auth-service` (and any future shared-cluster consumer) from the per-namespace bundled-Bitnami pattern to the shared CNPG `leartech-staging` cluster on jx-staging.

This is **mostly declarative** — the chart, helmfile, and ExternalSecret resources are all in git. Only the **secret values** themselves need to be set out-of-band by an admin, before the helmfile reconcile runs.

## One-time admin actions

Three secrets per cluster, populated via cloud CLI. Values are independent random 32-char strings.

### GCP cluster (`jx-build-cluster-gsm` / `tf-jx-usable-bird`)

```sh
gcloud config set project product-first

# 1. CNPG-managed role passwords (read by leartech-platform-postgres ExternalSecret).
openssl rand -base64 24 | tr -d '/+=' | head -c 32 | gcloud secrets create cnpg-auth-service-password --data-file=-
openssl rand -base64 24 | tr -d '/+=' | head -c 32 | gcloud secrets create cnpg-hydra-password         --data-file=-

# 2. Auth-service DSN — same secret name, new value pointing at CNPG.
#    Use whatever password from step 1 for auth_service.
AUTH_PW=$(gcloud secrets versions access latest --secret=cnpg-auth-service-password)
DSN="postgresql://auth_service:${AUTH_PW}@leartech-staging-rw.cnpg-system.svc.cluster.local:5432/auth_service?sslmode=require"
echo -n "$DSN" | gcloud secrets versions add auth-service-db-dsn --data-file=-
# (Or `gcloud secrets create` if the secret doesn't exist yet.)

# 3. Test user (staging-only, gated on postgresql.seedTestUser=true).
cat <<JSON | gcloud secrets create auth-service-test-user --data-file=-
{"email": "test@leartech.com", "password": "Test123!"}
JSON
```

### AZ cluster (`jx-build-cluster-akv` / `modern-burro`)

Vault paths instead of GSM keys; same shape:

```sh
# 1. Role passwords.
vault kv put secret/auth/cnpg-auth-service-password password="$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)"
vault kv put secret/auth/cnpg-hydra-password         password="$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)"

# 2. Auth-service DSN.
AUTH_PW=$(vault kv get -field=password secret/auth/cnpg-auth-service-password)
DSN="postgresql://auth_service:${AUTH_PW}@leartech-staging-rw.cnpg-system.svc.cluster.local:5432/auth_service?sslmode=require"
vault kv put secret/auth/auth-service-db dsn="$DSN"

# 3. Test user.
vault kv put secret/auth/test-user email=test@leartech.com password=Test123!
```

## Verify secrets land in cluster

```sh
# GCP
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n cnpg-system get secret auth-service-credentials -o jsonpath='{.data.username}' | base64 -d   # → auth_service
kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird -n jx-staging  get secret auth-service-db -o jsonpath='{.data.dsn}' | base64 -d                  # → postgresql://...rw.cnpg-system...

# AZ — same query, --context=modern-burro
```

## Flip the staging helmfile overlay

In `jx-build-cluster-gsm/helmfiles/jx-staging/configs/leartech-auth-service.yaml` (and AZ equivalent):

```yaml
storeBackend: postgres
postgresql:
  useSharedCluster: true
  runMigrations: true
  seedTestUser: true   # staging only — production stays false
  secretName: auth-service-db   # existing ExternalSecret, value rotated above
  secretKey: dsn
```

Open as PR. On merge + reconcile:

1. CNPG `Cluster.spec.managed.roles[]` creates `auth_service` + `hydra` Postgres roles using the new secrets.
2. Auth-service chart's `Database` CRs ask CNPG to provision the `auth_service` + `hydra` databases owned by those roles.
3. Migration `Job` (post-install hook) waits for the DB to come up + applies `0001_initial.sql`.
4. Seed-test-user `Job` (post-install hook, weight=5) seeds the test user with bcrypt-hashed credentials from the `auth-service-test-user` secret.
5. Auth-service deployment rolls — reads DSN from the rotated `auth-service-db` secret, connects to CNPG.

## Validation

```sh
# Auth-service deployment up + reading from CNPG
kubectl -n jx-staging logs deploy/leartech-auth-service | grep -i postgres
# Login flow
open https://leartech-auth-ui-jx-staging.jx.leartech.com   # log in with seed creds
```

## Rollback

DSN value rotation is non-destructive (old Bitnami pod still running until a separate cleanup PR removes it). To roll back:

```sh
gcloud secrets versions add auth-service-db-dsn --data-file=- < <(echo -n "$OLD_BITNAMI_DSN")
# wait for ExternalSecret resync (~60s default) → roll the auth-service deployment
kubectl -n jx-staging rollout restart deploy/leartech-auth-service
```

## Cleanup (separate PR, days later)

Once jx-staging has been stable on CNPG for a sensible interval:

1. Drop the `auth-postgresql` Bitnami release from `helmfiles/jx-staging/helmfile.yaml`.
2. Delete leftover PVC + StatefulSet.
3. Remove the now-unused `auth.postgresql` block from `charts/auth-resources/values.yaml` if it references the Bitnami secret.

Preview namespaces are unaffected — they continue using the per-namespace ephemeral Bitnami via `preview/postgresql.yaml`.
