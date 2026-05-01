# HAR pipeline — capture, sanitize, replay

Strategic infrastructure: HAR (HTTP Archive) as a multi-producer, multi-consumer interchange format. Multiple sources produce HAR (Playwright today, Tempo Phase 2, optionally Hubble/Pixie Phase 3); a single sanitize/template layer cleans them; a single replay engine consumes them. Load testing is the first consumer; test generation, contract derivation, coverage reporting are future cheap additions.

This is the design that breaks mqube's coupling-of-load-testing-to-Playwright while keeping the elegant HAR-replay pattern they pioneered.

## Three-tier pipeline

```
PRODUCERS                    SANITIZE/TEMPLATE                CONSUMERS
─────────                    ─────────────────                ─────────

Playwright HAR            ┌─ strip auth headers          ─┐  
   (from leartech-qa)     │  (Authorization,              │  
                          │   Cookie, Set-Cookie,         │  
                          │   X-API-Key, etc.)            │  
Tempo→HAR synthesizer     │                                │  leartech-load-testing
   (Phase 2; reads        │  redact PII patterns          │     (HAR replay engine,
    OTel spans; no new    │  (emails, tokens-in-URLs,     │      OAuth-substituted,
    infra)                │   phone numbers via regex)    │      SLA-asserted)
                          │                                │
Hubble→HAR (Phase 3)      │  parameterize ephemeral       │  test-generation AI
   (eBPF; only if Tempo   │  IDs:                         │     (proposes Playwright
    coverage gaps)        │  /api/cases/{{ caseId }}      │      additions for high-traffic
                          │  /api/users/{{ userId }}      │      untested edges)
Pixie→HAR (Phase 3 alt)   │                                │
   (no CNI swap)          │  store as canonical HAR       │  contract-derivation tool
                          │  in GCS, indexed by source +  │     (Pact contracts from
gateway-log→HAR           │  service + timestamp          │      observed pairs)
   (if a gateway exists   │                                │
    that logs requests)   │                                │  coverage-gap reporter
                          │                                │     (HAR-source diff)
OpenAPI synth→HAR         └─                             ─┘
   (schema-derived;                                          security-replay (mutated
    fills gaps in real                                        payloads / fuzzing)
    traffic)
```

## Why HAR specifically

- **Standard format** — `{ log: { entries: [{ request, response, timings }] } }`. Browsers, Playwright, OWASP ZAP, Postman all speak it.
- **Lossless enough** — captures URL, method, headers, body, query, status, response, timings.
- **Reasonable size** — JSON; compresses well; per-test HARs are typically 10-500KB.
- **Tooling exists** — Go libraries (`hargo`, used by mqube's sanitize), JS (`har-validator`), Python (`haralyzer`).

Alternatives considered:
- **OpenAPI** — too abstract; loses real traffic shapes
- **Postman collections** — proprietary; conversion is lossy
- **Custom format** — reinvents wheel, locks consumers in

HAR wins.

## Producers

### 1. Playwright HAR (Phase 1 — already partly free)

Leartech's existing `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml` already runs Playwright against previews and uploads `.zip` traces (which contain HAR-equivalent data) to `gs://test-artifacts-product-first/`. The mechanism + auth is settled.

Two small extensions:

1. **Configure Playwright to record HAR explicitly** alongside traces — set `RecordHarPath` in the test runner config. Same as mqube's `PlaywrightService.cs:45`. ~half day.
2. **Extend the upload to push the HAR to a HAR-specific path** (the existing path is artifact-keyed; HAR replay wants a HAR-keyed path):
   ```
   gs://test-artifacts-product-first/har/v1/<repo>/<sha>/<run-id>/trace.har
   ```
   ~1 hour.

**Cost**: ~half day total. The bucket, auth, and upload pattern already exist.

### 2. Tempo→HAR synthesizer (Phase 2)

A small Go service (`tempo-to-har`) that:

1. Queries Tempo's HTTP API for spans in a window (e.g. last 24h, filter by service)
2. Groups spans by trace ID — each trace ≈ one user request flow
3. For each span representing an HTTP call, extracts: URL, method, headers (from span attributes), status, duration
4. Synthesizes a HAR file with those entries
5. Pushes to GCS (`gs://leartech-qa-har/v1/tempo-synth/<window>/<trace-id>.har`)

**Why this works for leartech**: golden-service-standard mandates OTel emission. Coverage = mandate adherence. Should be high.

**Limitation**: no request bodies in OTel spans by default (they're not captured for cost/PII reasons). For load replay, body-less HAR works for GET/DELETE but limits POST/PUT testing to schema-derived bodies (or skip those). For coverage-gap reporting and test generation, body-less HAR is sufficient.

**Cost**: ~3 days (Tempo HTTP client + grouping + HAR templating in Go).

### 3. Hubble or Pixie→HAR (Phase 3, conditional)

Only build if Tempo gaps are real (services without OTel, third-party traffic, raw TCP etc.).

- **Cilium Hubble** — eBPF, requires Cilium CNI. Real cluster change. Best long-term if you also want network policy / mTLS via the same investment.
- **Pixie** — eBPF, doesn't require CNI swap. Lighter commitment. Has its own query language (PxL).

Either way, build a `<tool>-to-har` exporter that runs on a schedule and writes to GCS. Same target shape as Tempo synthesizer.

**Cost**: 1-2 weeks for Pixie deploy + exporter; weeks-to-months for Cilium CNI migration + Hubble exporter.

### 4. Gateway log → HAR (Phase 2 if applicable)

If leartech has an API gateway (Ocelot equivalent of mqube's mqube-apiGW, or similar), and it emits structured access logs to Loki, those logs can be reformatted to HAR.

**Cost**: ~1 day if logs are structured; longer if you have to rebuild from text logs.

### 5. OpenAPI synth → HAR (Phase 2 supplement)

For services where neither Playwright nor Tempo provide good coverage of an endpoint (e.g. an admin endpoint nobody tests), synthesize HAR from the OpenAPI spec:

- Generate request examples per endpoint (use the OpenAPI `examples` field where present, or schema-derived)
- Wrap as HAR
- Push to GCS

Lower fidelity than real traffic but fills coverage holes deterministically.

**Cost**: ~2 days (Go OpenAPI library + templating).

## Sanitize / template layer

Single purpose: take any HAR from any producer, output a HAR safe + ready for replay.

```go
type Sanitizer interface {
    Sanitize(har *HAR, opts SanitizeOptions) (*HAR, error)
}

type SanitizeOptions struct {
    StripAuth         bool   // default true
    RedactPII         bool   // default true
    Parameterize      []ParameterizationRule  // {{ caseId }} substitutions
    PreserveBodies    bool   // default true; false if storing for diff/test-gen only
}

type ParameterizationRule struct {
    Pattern    *regexp.Regexp  // e.g. `/api/cases/[a-f0-9-]{36}`
    Replacement string          // e.g. `/api/cases/{{ caseId }}`
}
```

Two phases:

### Phase A: redact (storage-time)

Run by every producer before pushing to GCS. Removes:

- Auth headers (Authorization, X-API-Key, Cookie, Set-Cookie, etc.)
- Auth tokens in query strings (?token=...)
- Bearer token patterns in any header value
- Email addresses (regex)
- PII patterns (phone numbers, SSNs, credit-card numbers — configurable)

mqube's `cmd/sanitize/main.go` is forkable. ~2 days to adapt for leartech.

### Phase B: template (replay-time)

Run by consumers when reading HAR. Substitutes:

- Auth: replace placeholder Authorization header with fresh OAuth client-credentials token (minted at run time)
- Parameterize: replace `{{ caseId }}` etc. with run-time values from a values file or service call

This is what mqube-load-testing does at replay (mints token via `MQUBE_AUTH_CLIENT_ID/SECRET` env). Lift the pattern.

## Replay engine — `leartech-load-testing`

Direct lift of mqube-load-testing's pattern. Go service, Tekton-deployable. Differences from mqube:

| Concern | mqube | This proposal |
|---|---|---|
| HAR source | Playwright only (via automated-qa-service) | Multi-producer (Playwright, Tempo, etc.) |
| Auth handling | OAuth at replay (good) | Same |
| SLA assertions | None observable | Built in — reads `load/<service>.yaml` from qa-management |
| Trigger | Unknown (probably CronJob curl) | CronJob nightly + Tekton step in promotion gate |
| Result sink | Mongo `Mqube-LoadTesting.LoadTestingResults` + Tempo | Same: Mongo + Tempo |
| Gate integration | None (mqube doesn't have a load-quill) | `load-sla` quill in leartech-gate reads results, fails on regression |

Service exposes:

- `POST /api/loadtest/run/{harId}` — replay a specific HAR
- `POST /api/loadtest/run-latest/{service}` — latest HAR from default producer for a service
- `GET /api/loadtest/result/{runId}` — fetch result + SLA verdict
- `/health`, `/metrics`, `/swagger` — standard

**Build estimate** (with leverage): ~3 days for the Go service skeleton + replay logic + auth substitution + result write. ~2 days for the SLA assertion engine + quill integration.

## Result schema (replay output)

```json
{
  "schema_version": "v1",
  "run_id": "...",
  "har_id": "...",
  "har_source": "playwright",
  "service": "leartech-broker-ui",
  "started_at": "2026-04-29T02:00:00Z",
  "duration_seconds": 1800,
  "summary": {
    "total_requests": 125000,
    "successful": 124850,
    "failed": 150,
    "p50_ms": 120,
    "p95_ms": 480,
    "p99_ms": 1200,
    "error_rate": 0.0012
  },
  "sla_verdict": {
    "passed": true,
    "checks": [
      {"name": "p95_latency_ms < 500", "actual": 480, "passed": true},
      {"name": "error_rate < 0.01", "actual": 0.0012, "passed": true},
      {"name": "p95 regression < 10%", "baseline_ms": 470, "regression_pct": 2.1, "passed": true}
    ]
  },
  "trace_url": "https://grafana.leartech.build/explore?...",
  "tempo_trace_id": "abc123def"
}
```

Written to result-store (same store the gate reads). Consumed by `load-sla` quill on next promotion PR.

## Triggers

### Nightly CronJob per service

```yaml
# k8s CronJob in jx-staging
apiVersion: batch/v1
kind: CronJob
metadata: { name: leartech-loadtest-broker-ui-nightly }
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trigger
            image: curlimages/curl:latest
            command:
              - /bin/sh
              - -c
              - |
                curl -X POST \
                  -H "Authorization: Bearer $TOKEN" \
                  https://leartech-load-testing-jx-staging/api/loadtest/run-latest/leartech-broker-ui
```

One CronJob per service to load-test. Generated from `load/<service>.yaml` entries in qa-management — a small Go script (`load-cronjob-gen`) reads the configs and emits the K8s YAML to the GitOps repo. Renovate-style PR when SLA configs change.

### Tekton step in promotion gate (Phase 2)

For services where load testing must gate (perf-critical paths), the promotion-PR pipeline runs the load test inline:

```yaml
- image: leartech-load-testing-cli:1.0.0
  name: load-sla-check
  script: |
    leartech-load-testing run-and-wait --service leartech-broker-ui --sha $PR_SHA
    # exits non-zero if SLA failed
```

The result lands in result-store; the `load-sla` quill in leartech-gate reads it.

## Future consumers (cheap to add once HAR pipeline exists)

### Risk-assessor (Phase 2.5)

The HAR pipeline + Tempo→HAR producers feed `leartech-risk-assessor` as **confirming signals** for risk classification. See `risk-assessor.md` for the full design — HAR endpoint extraction + Tempo span breadth become two of the inputs that distinguish "PR's preview just ran narrow tests" from "PR genuinely has narrow scope". This is the most concrete near-term consumer beyond load testing.

### Test-generation AI

```
Inputs: 
  - Tempo→HAR (production traffic shapes)
  - Playwright HARs (test coverage)
Diff: edges in production, not in tests
Per untested edge:
  - Compose a Playwright test stub
  - Open PR to relevant -ui or -service repo
  - AI code reviewer reviews + author refines
```

**Cost**: ~3 days using leartech-ai-classifier infra.

### Contract derivation

```
Inputs: production HAR (request + response pairs)
Per service-pair edge:
  - Extract request schema (from request body)
  - Extract response schema (from response body)
  - Generate Pact contract YAML
  - Open PR to consumer repo's pact/ dir
```

**Cost**: ~1 week if Pact tooling is new to the org; ~3 days if Pact is already in use.

### Coverage-gap reporter

The Hubble/Tempo vs. Playwright diff. Outputs a weekly "untested edges" report that becomes input to test-generation.

**Cost**: ~2 days.

### Security replay (basic fuzzing)

For each HAR entry, mutate (random byte flips in body, header injection, SQLi/XSS payloads) and replay against staging. Watch for 5xx / unexpected behavior. Lightweight DAST that doesn't need ZAP integration.

**Cost**: ~1 week.

## Storage estimate

Per-test Playwright HAR: ~50KB-500KB compressed. Assume ~1000 test runs/day × 200KB avg = 200MB/day = ~73GB/year. GCS at ~$0.02/GB/month = ~$18/year for HAR storage. Negligible.

Tempo→HAR synthesized output similar volume. Hubble/Pixie if added: orders of magnitude more (sample heavily).

Lifecycle policy: 90 days retention default; longer for HARs flagged as "interesting" (failures, perf outliers).

## What this design buys you over mqube

- **Multiple producers** = Playwright coverage gaps don't blind load testing
- **HAR as strategic asset** = future consumers (test-gen, contract, coverage-gap) are cheap additions
- **SLA-aware replay** = load testing can actually gate (mqube's doesn't)
- **Single sanitize layer** = each producer doesn't reinvent redaction; new sources plug in cleanly
- **Tempo-first mapping** = leverages observability stack you already have; defers eBPF infra commitment

## Open decisions for this component

See `open-questions.md`. Highlights:

- Body-stripping policy: keep request bodies in HAR (more useful, more PII risk) or strip (safer, less powerful for replay)?
- Long-term storage: 90 days default OK, or longer for compliance reasons?
- Producers' priority order: Playwright + Tempo are clear Phase 1+2; gateway logs vs OpenAPI synth — which third source first?
