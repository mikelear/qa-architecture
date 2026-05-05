# Session 2.4 — `tempo-to-har` synthesizer

Build a Go service that queries Tempo's HTTP API for span data, synthesizes HAR (HTTP Archive) files describing the captured network traffic, and uploads them to GCS. Establishes the foundation for both load-testing (Phase 2.2) and traffic-forensics (Phase 2.7).

This is the first **consumer** of Tempo data the leartech QA system has. The Go canary already emits OTel spans (per golden-service-standard mandate); Tempo already stores them; this session unlocks them as a first-class data source for the rest of the architecture.

**Time budget**: 3-4 hours.

---

## Why this session jumps the queue

Tempo is the highest-leverage data source we haven't tapped yet:

- **Already running** on both clusters (`tempo.jx-observability.svc.cluster.local:4317`)
- **Already populated** — every golden-template-Go service's HTTP calls land there as OTel spans
- **HAR is a well-defined format** — Phase 2 load-testing + Phase 2.7 traffic-forensics both consume it
- **Stable HTTP API** — Tempo's `/api/search` + `/api/traces/<id>` are versioned and well-documented

So we get a usable Tempo→HAR pipeline immediately. Future phases (load-testing, forensics) become "thin consumer of HAR" rather than "stand up Tempo + write a synthesizer + ...".

---

## Outcome we're proving

```
Canary (or any golden-template-Go service) handles a request
       ↓ (OTel SDK in leartech-go-common)
spans flow to Tempo via OTLP
       ↓
tempo-to-har synthesizer queries Tempo periodically (or on demand)
       ↓
extracts spans for a given service / time window
       ↓
groups by trace ID; synthesizes HAR-format JSON
       ↓
uploads to GCS at gs://test-artifacts-product-first/har/v1/<source>/<service>/<window>.har

Future consumers (Phase 2 + 2.7):
  - load-testing service replays the HAR against a target env
  - arrivals-observer's forensics engine diffs HARs between version windows
  - test-generation AI proposes Playwright tests for unobserved edges
```

---

## Pre-flight checklist

- [ ] Tempo reachable from cluster — `kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get svc -n jx-observability | grep tempo`
- [ ] Tempo has data — query the canary's traces from Session 0:

  ```bash
  TEMPO_HOST=$(kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird get svc tempo-query-frontend -n jx-observability -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  # or use a port-forward:
  kubectl --context=gke_product-first_us-east1-b_tf-jx-usable-bird port-forward -n jx-observability svc/tempo-query-frontend 3200:3200 &
  
  # Quick sanity: does Tempo know about leartech-qa-canary?
  curl -s "http://localhost:3200/api/search?tags=service.name%3Dleartech-qa-canary&limit=5" | jq '.traces | length'
  # Expect: > 0 (canary's end2end runs left traces)
  ```

- [ ] `gh auth status` — `mikelear` authenticated
- [ ] Session 0 bootstrap fixes applied to runbook (lessons.md items 1-4 — we'll use them when creating tempo-to-har)
- [ ] golden Go template still at `~/leartech/leartech-go-service-template/`

---

## Deliverables

### Step 1 — create `mikelear/tempo-to-har` from golden template

Following the **corrected** clone-and-rename procedure from `session-0-lessons.md` items 1-2:

```bash
gh repo create mikelear/tempo-to-har --public \
  --description "Synthesizes HAR (HTTP Archive) files from Tempo OTel spans. Foundation data source for the leartech QA system's load-testing (Phase 2.2) and traffic-forensics (Phase 2.7) consumers. See ~/leartech/qa-architecture/har-pipeline.md for design."

cd /tmp && rm -rf tempo-to-har 2>/dev/null
git clone --depth=1 https://github.com/mikelear/leartech-go-service-template.git tempo-to-har
cd tempo-to-har && rm -rf .git

# Comprehensive sed (both module path AND bare name)
OLD_FQ=github.com/mikelear/leartech-go-service-template
NEW_FQ=github.com/mikelear/tempo-to-har
OLD_BARE=leartech-go-service-template
NEW_BARE=tempo-to-har
find . -type f \( -name '*.go' -o -name '*.yaml' -o -name '*.yml' -o -name '*.json' -o -name '*.md' -o -name 'go.mod' -o -name 'Makefile' -o -name 'Dockerfile' \) \
  -not -path './.git/*' -not -path './.angular/*' \
  -exec sed -i.bak "s|$OLD_FQ|$NEW_FQ|g; s|$OLD_BARE|$NEW_BARE|g" {} \; -exec rm -f {}.bak \;

# Chart directory rename (gap #2 from Session 0)
mv charts/leartech-go-service-template charts/tempo-to-har

# DB-disable (gap #3 from Session 0) — tempo-to-har has no Postgres need
sed -i.bak '/^database:/,/^migrations:$/{s|^  enabled: true$|  enabled: false|}' charts/tempo-to-har/values.yaml
sed -i.bak '/^migrations:/,/^[a-z]/{s|^  enabled: true$|  enabled: false|}' charts/tempo-to-har/values.yaml
rm -f charts/tempo-to-har/values.yaml.bak*

# Verify clean
go mod tidy && go build ./... && yq eval '.' charts/tempo-to-har/Chart.yaml >/dev/null
grep -rln leartech-go-service-template . 2>/dev/null | grep -v "\.git/\|\.angular/" | head
# Expected: empty

git init -q -b main
git remote add origin https://github.com/mikelear/tempo-to-har.git
git add .
git commit -m "Initial commit — tempo-to-har scaffold"
git push -u origin main
```

### Step 2 — register in source-config (both clusters)

Add to both `~/leartech/jx-build-cluster-gsm/.jx/gitops/source-config.yaml` and `~/leartech/jx-build-cluster-akv/.jx/gitops/source-config.yaml`:

```yaml
- name: tempo-to-har
  description: "Synthesizes HAR files from Tempo OTel spans. Foundation data source for leartech QA system's load-testing + traffic-forensics consumers"
```

Push to both. Wait for git-operator reconcile (<5min from Session 0 lesson #8).

### Step 3 — implement Tempo client

`internal/tempo/client.go` — minimal HTTP client wrapping the two endpoints we need:

```go
// Search returns trace IDs matching tags within a window.
// Tempo /api/search supports: tags=service.name=<name>, start=<unix>, end=<unix>, limit=<n>
func (c *Client) Search(ctx context.Context, serviceName string, since, until time.Time, limit int) ([]TraceRef, error)

// Trace returns the full span tree for a trace ID.
// Tempo /api/traces/<id> returns OTLP-format spans.
func (c *Client) Trace(ctx context.Context, traceID string) (*Trace, error)
```

Tempo's HTTP API spec: <https://grafana.com/docs/tempo/latest/api_docs/>

Minimum span fields we need:
- `name` — operation name (often the HTTP route)
- `attributes` — http.method, http.url, http.status_code, http.request.body.size, http.response.body.size
- `start_time` / `end_time` — for HAR `startedDateTime` + `time` in ms
- `parent_span_id` — to identify root vs child spans

### Step 4 — span-to-HAR converter

`internal/synth/har.go` — converts a list of HTTP-flavored spans into HAR JSON.

HAR spec: <http://www.softwareishard.com/blog/har-12-spec/>

Minimum HAR fields per entry:

```json
{
  "startedDateTime": "2026-05-05T13:00:00.000Z",
  "time": 1234,
  "request": {
    "method": "GET",
    "url": "https://...",
    "httpVersion": "HTTP/1.1",
    "headers": [],
    "queryString": [],
    "cookies": [],
    "headersSize": -1,
    "bodySize": 0
  },
  "response": {
    "status": 200,
    "statusText": "OK",
    "httpVersion": "HTTP/1.1",
    "headers": [],
    "cookies": [],
    "content": { "size": 0, "mimeType": "" },
    "redirectURL": "",
    "headersSize": -1,
    "bodySize": 0
  },
  "cache": {},
  "timings": { "send": 0, "wait": 1234, "receive": 0 }
}
```

Fields we DON'T have from Tempo (and so emit empty / minus-one):
- request body content (Tempo doesn't capture by default)
- response body content (same)
- exact headers (only what's in span attributes, e.g. `http.url` includes path+query)

That's fine — load-testing replay gets URL + method + path + status; forensics gets edges + rates + latencies. Body content is **not** required for either consumer in their initial form.

### Step 5 — CronJob + on-demand HTTP API

The service is dual-mode:

**CronJob mode** — runs on a schedule (e.g. every hour). For each `service` listed in config:
- Query Tempo for traces in the last hour
- For each trace, fetch the full span tree
- Synthesize HAR (one HAR per service per hour-window OR one HAR per trace, TBD)
- Upload to `gs://test-artifacts-product-first/har/v1/tempo-synth/<service>/<window>.har`

**HTTP API mode** — for ad-hoc / Phase 2.7 use:
- `GET /api/synth?service=<name>&since=<unix>&until=<unix>` → triggers a synth run, returns the GCS URL of the resulting HAR

Initial spike scope: **CronJob only**. HTTP API is Phase 2.5+ when arrivals-observer's forensics engine wants on-demand access.

### Step 6 — config-driven service list

`charts/tempo-to-har/values.yaml` adds a `services:` list:

```yaml
services:
  - name: leartech-qa-canary
    har_window_minutes: 60
    har_filter:
      include_methods: [GET, POST, PUT, DELETE, PATCH]
      exclude_paths: ["^/health/", "^/metrics$"]
  # add more services as adoption grows
```

Mounted as ConfigMap; service reads at startup. No restart needed when config changes; CronJob picks up fresh values per run.

### Step 7 — validate against canary's real traces

After deploying the CronJob in jx-staging, wait for one cycle:

```bash
gcloud storage ls gs://test-artifacts-product-first/har/v1/tempo-synth/leartech-qa-canary/

# Inspect the produced HAR
gcloud storage cat gs://test-artifacts-product-first/har/v1/tempo-synth/leartech-qa-canary/<latest>.har | jq '.log.entries | length'
# Expect: > 0 (every health check probe + e2e smoke run leaves traces)

# Inspect a single entry
gcloud storage cat gs://test-artifacts-product-first/har/v1/tempo-synth/leartech-qa-canary/<latest>.har | jq '.log.entries[0]'
# Expect: shape matches HAR spec; URL + method + status all present + reasonable
```

### Step 8 — capture lessons

Update `~/leartech/qa-architecture/session-0-lessons.md` (or split into a Phase-2 lessons file) with anything we encounter:

- Tempo HTTP API quirks (auth pattern, port-forward vs LB, query DSL surprises)
- Span volume vs window size — how much HAR per minute of canary traffic?
- HAR validation tooling — does `har-validator` accept the synthesized output?
- Memory/CPU resource floor for the synthesizer

---

## Validation criteria

- [ ] `mikelear/tempo-to-har` repo exists, builds clean, all bootstrap fixes applied (zero residual references; chart-dir renamed; DB-disabled)
- [ ] Source-config registered on both clusters; webhooks installed
- [ ] First-PR pipelines pass (using the corrected runbook from Session 0; <20 min wall-clock)
- [ ] Release pipeline pushes image to registry (verify Session 0c lessons applied)
- [ ] CronJob deployed in jx-staging on at least one cluster
- [ ] At least one HAR file written to GCS at `har/v1/tempo-synth/leartech-qa-canary/`
- [ ] HAR is valid per HAR 1.2 spec (`har-validator` clean OR manual schema check)
- [ ] HAR entries reflect canary's actual traffic (URLs, methods, statuses match what end2end smoke saw)
- [ ] `session-0-lessons.md` updated with any new findings

---

## What we deliberately SKIP

- **Sanitize / template layer** — `notifications.md` Phase 2 design has this; tempo-to-har v1 emits raw HAR. Sanitize is built once across multiple producers; defer until the HAR pipeline has a second producer.
- **HTTP API mode** — CronJob only. On-demand API is Phase 2.5+ when arrivals-observer needs it.
- **Cross-cluster fan-out** — single cluster (gcp) deploy. Az parity is Phase 1.5-style hardening.
- **Authenticated Tempo queries** — assume in-cluster Tempo is reachable without auth (it is). Cross-cluster queries need auth; Phase 1 hardening.
- **Multi-trace correlation** — each trace becomes its own HAR. Cross-trace patterns (e.g. "all traces for a given user session") deferred.
- **Backfill mode** — only forward (going-from-now). Phase 2.7 might want backfill for forensics-after-the-fact.

---

## Anti-scope-creep

- "Let me also wire load-testing service to consume the HARs" → **No.** Phase 2.2.
- "Let me build the forensics diff engine" → **No.** Phase 2.7.
- "Let me add Pixie/Hubble as a second producer" → **No.** Phase 3 conditional.
- "Let me write tests against real Tempo data" → One smoke test fine; don't go deep until Phase 1 hardening.
- "Let me add the sanitize layer now" → **No.** Single producer for v1; sanitize when adding a second.

---

## Anticipated failure modes

| Failure | Likely cause | Fix |
|---|---|---|
| Tempo `/api/search` returns 0 traces despite canary running | Tags filter mismatch — Tempo uses `service.name` as resource attribute; canary may emit it as `service_name` or similar | Read canary's OTel SDK config to confirm attribute key; adjust query |
| Tempo HTTP API returns 401 | tempo-query-frontend has gateway-auth in front | port-forward bypasses; CronJob in-cluster needs no auth (same namespace?) — verify |
| HAR JSON malformed (har-validator complains) | Required field missing (most often: `version`, `creator`, root structure) | Check examples/ in canary's chart for any reference HAR format; conform to HAR 1.2 strictly |
| Span volume too high for memory | Each `/api/traces` call returns full span tree which can be MB; iterating all traces in a window is unbounded | Implement pagination + per-trace memory bound; cap at 100 traces per service per window |
| GCS upload 403 | Service account lacks `roles/storage.objectCreator` on the bucket | Add IAM binding; same secret pattern as end2end task uses |

---

## Reading order

1. `~/leartech/qa-architecture/sessions.md` (live-status)
2. `~/leartech/qa-architecture/session-0-lessons.md` — bootstrap runbook (use it!)
3. `~/leartech/qa-architecture/har-pipeline.md` — overall HAR pipeline design
4. `~/leartech/qa-architecture/notifications.md` — for the future-consumer Notifier integration
5. <https://grafana.com/docs/tempo/latest/api_docs/> — Tempo API spec
6. <http://www.softwareishard.com/blog/har-12-spec/> — HAR 1.2 spec
7. This brief

That's it. Get coding.
