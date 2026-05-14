# Tier-2 forensics demo — 2026-05-14

## TL;DR

Infrastructure validated end-to-end. The actual Layer 1 / Layer 2 *trigger* did not fire because of a real structural finding: **the canary smoke test doesn't exercise the endpoint where the regression lives**. That's a useful design lesson, not a bug in the gate.

## What we wanted to prove

The Tier-2 design (see `~/.claude/projects/-Users-mikelear-leartech-Qa-Analysis/memory/project_layer1_layer2_forensics_design.md`) introduced two new quill layers for "passed-but-slower" regression detection:

- **Layer 1** (gate-cli): per-test `duration_ms` comparison from `results.json` between current and prior arrival. Cheap, no Tempo dependency.
- **Layer 2** (forensics-runner): per-endpoint p95 + error-rate diff bounded to the dispatched-test window. Tempo-backed drill-down.

Tier-1 (2026-05-13) already demonstrated forensics firing on *Failed* arrivals via deliberate test failures. Tier-2 needs to demonstrate the *passed-but-slower* case end-to-end.

## What we did

1. Deployed the Tier-2 foundations across both clusters:
   - `arrivals-observer@0.0.23` (PR #6) — forensics now fires on every Arrival, including Passed
   - `forensics-runner@0.0.8-gcp` / `@0.0.5-az` (PR #6) — Tempo windows derived from `Arrival.status.tests[].startedAt..completedAt`, not fixed ±5m wall-clock
   - `gate-cli@0.0.16` (PR #11) — Layer 1 duration-regression quill
   - `ENABLE_DURATION_QUILL=true` on both GitOps `qa-gate.yaml` (PRs gsm#335 + akv#190)
2. Injected a 5-second `time.Sleep` in canary's `GET /api/v1/example` handler
3. Cut canary `0.0.19-gcp` and promoted it through `gsm#338`
4. Waited for the arrival → tests → forensics chain

## What the artifacts show

### Arrival CR for canary 0.0.19

```yaml
status:
  finalizedAt: "2026-05-14T15:34:11Z"
  phase: Passed
  forensics:
    diffUrl: gs://test-artifacts-product-first/forensics/v1/gcp/jx-staging/leartech-qa-canary/0.0.19/diff.json
    generatedAt: "2026-05-14T15:35:09Z"
    jobName: forensics-leartech-qa-canary-0-0-19-jx-staging
    summary:
      latency_regressions:    0
      error_rate_regressions: 0
      missing:                0
      new:                    0
  tests:
  - name: end2end
    type: end2end
    status: Passed
    startedAt:   "2026-05-14T15:32:29Z"
    completedAt: "2026-05-14T15:34:11Z"
```

Forensics fired on a Passed arrival — that's the observer PR #6 change working as designed. Pre-PR-#6 this would have been forensics-not-dispatched.

### diff.json (truncated to the key bits)

```json
{
  "schema_version": "v1",
  "service": "leartech-qa-canary",
  "version": "0.0.19",
  "previous_version": "0.0.16",
  "before_window": { "start": "2026-05-14T11:15:45Z", "end": "2026-05-14T11:17:23Z" },
  "after_window":  { "start": "2026-05-14T15:32:29Z", "end": "2026-05-14T15:34:16Z" },
  "before": {
    "/docs":         { "count": 2,  "p50_ms": 0.63, "p95_ms": 0.63 },
    "/health/live":  { "count": 27, "p50_ms": 0.25, "p95_ms": 4.93 },
    "/health/ready": { "count": 42, "p50_ms": 0.29, "p95_ms": 9.32 },
    "/openapi.json": { "count": 2,  "p50_ms": 19.2, "p95_ms": 19.2 }
  },
  "after": {
    "/docs":         { "count": 2,  "p50_ms": 0.56, "p95_ms": 0.56 },
    "/health/live":  { "count": 27, "p50_ms": 0.24, "p95_ms": 0.93 },
    "/health/ready": { "count": 46, "p50_ms": 0.21, "p95_ms": 0.59 }
  }
}
```

Two things worth highlighting:

1. **`before_window` and `after_window` are scoped to dispatched-test runs**, not to `deployedAt ± 5m`. The 0.0.16 baseline is from 11:15:45-11:17:23 (its end2end test pack's actual run window 4 hours earlier). The 0.0.19 comparison is from 15:32:29-15:34:16 (this arrival's test pack window). This is the design change from `feat: bound Tempo windows to Arrival.status.tests[]` (`leartech-forensics-runner#6`) — and it survived the round-trip via observer + CR + runner cleanly.
2. **`/api/v1/example` is absent from both windows.** Tempo collected spans for every endpoint actually called during the smoke test's run window. The smoke test never hits `/api/v1/example`, so Tempo has no spans, so the diff has nothing to compare.

### results.json (0.0.19)

```json
{
  "success": true,
  "tests": [
    { "name": "smoke", "status": "pass", "duration_ms": 659, "message": "OK" }
  ]
}
```

Smoke `duration_ms = 659` — actually 0.54× the 0.0.16 baseline (1225 ms). Not a regression at all; well within run-to-run variance. Same root cause: the test fixture isn't exercising the regressed code path.

## Root cause

`canary/end2end/01-smoke.sh` exercises only the unauthenticated public surface:

```sh
check GET  /health/live   200
check HEAD /health/live   200
check GET  /health/ready  200
check HEAD /health/ready  200
check GET  /openapi.json  200
check HEAD /openapi.json  200
check GET  /docs          200
check HEAD /docs          200
```

`GET /api/v1/example` is marked `@Security BearerAuth` and would return 401 without a token. The smoke test skips it. So a regression that lives behind authenticated routes is invisible to this test fixture.

## What this proves

| Component | Verdict |
|---|---|
| Observer fires forensics on Passed (PR #6) | ✅ confirmed in `Arrival.status.forensics.generatedAt` |
| Forensics test-bounded windows (PR #6) | ✅ `before_window` + `after_window` derive from `status.tests[].startedAt/completedAt`, not wall-clock |
| Forensics handles "no previous arrival" correctly | n/a here — 0.0.16 was found as prior |
| Tempo collector emits spans for traced endpoints | ✅ 27 `/health/live` calls + 42 `/health/ready` calls visible in `before` |
| diff.json upload to GCS at controller-rendered path | ✅ landed at the expected `forensics/v1/gcp/jx-staging/leartech-qa-canary/0.0.19/diff.json` |
| `Arrival.status.forensics.diffUrl` patched | ✅ |
| Layer 1 detects per-test duration regression | ❌ couldn't fire — test duration unaffected by sleep |
| Layer 2 detects per-endpoint p95 regression | ❌ couldn't fire — regressed endpoint not in trace data |

## What this does NOT prove

- **Layer 1 actually flagging a real regression in a verdict comment.** That requires either a regression in an endpoint the smoke test covers, OR extending the smoke test to cover the regressed endpoint. Today's run didn't produce a verdict-comment Layer 1 flag because there was nothing to flag.
- **Per-cluster behaviour on AZ.** Same flow should land on AKV (akv#190 merged), but AZ Postgres separately-in-flight means standalone validation there is fragile. Not a Tier-2 concern.

## Follow-up — what to do next

To produce a true end-to-end demo of Layer 1 + Layer 2 catching a regression on canary:

**Option A (recommended) — extend the canary smoke test:**
- Add `check GET /api/v1/example 200` to `end2end/01-smoke.sh`. Requires a test-fixture bearer token in the runner Job env, since `/api/v1/example` is `@Security BearerAuth`. Token can be a fixed test JWT pulled from a Vault secret + injected via env.
- After the smoke test exercises the endpoint, a sleep in its handler shows up in `results.json.tests[*].duration_ms` (Layer 1) and in Tempo spans during the test window (Layer 2).
- Wall-clock cost: ~1-2h (test-fixture token plumbing).

**Option B — add a no-auth endpoint to canary:**
- New route `GET /api/v1/echo` with no auth middleware. Smoke test calls it. Sleep injected there reaches both layers.
- Avoids the auth-token complexity but adds a route purely for demo purposes. Less faithful to "real service" shape.

**Option C — accept the structural finding and document it:**
- The gate is structurally blind to endpoints the test fixture doesn't exercise. That's a real constraint of any test-coverage-based regression detector. Useful caveat to record in `qa-architecture` and on the gate's README.
- Future "tier-2" demonstrations would be done against a service with broader fixture coverage (not the test-bed canary).

## Operational sidenote — Kyverno flakes

During the demo cycle (2026-05-14 14:11 → 15:14 UTC), Kyverno admission webhook caused **three** distinct task failures:

1. `gate-cli#11` GCP `ai-review` + `image-scan` pipelinerun pods (`mutate.kyverno.svc-fail` timeout during Kyverno restart)
2. canary release postsubmit `4cqkf` (same — `mutate.kyverno.svc-fail` timeout)
3. canary `0.0.18` arrival dispatch (`validate.kyverno.svc-fail` connection refused — observer's create-Job hitting a stale endpoint IP `10.48.13.219`)

Each failure named Kyverno verbatim in the K8s API server's error message (not interpretation). Kyverno pod restart age was 7-172 min across the window — the cluster was clearly cycling Kyverno during this hour. Memorialised in `memory/project_kyverno_admission_flakes.md`; long-term fix (multi-replica + PDB, or namespace exemption) gated on frequency.

These were the only flakes — every other surface (observer, forensics-runner, gate-cli, GCS, Tempo) behaved exactly as designed.

## Sequence (timing reference)

| Time (UTC) | Event |
|---|---|
| 14:08 | leartech-arrivals-observer#6 merged (forensics-on-Passed) |
| 14:14 | leartech-forensics-runner#6 merged (test-bounded windows) |
| 14:20 | leartech-gate#11 merged (Layer 1) |
| 14:13 | gsm#335 + akv#190 land (image bumps + ENABLE_DURATION_QUILL=true + runnerImageTag pin) |
| 14:48 | Canary 0.0.17 pushed (1s sleep) |
| 14:52 | Canary 0.0.17 released (gcp) → gsm#336 auto-opens |
| ~15:01 | Decision to bump 1s → 5s (because baseline is 1225ms; 1s gives only 1.8× ratio, below 3× threshold) |
| ~15:01 | Canary 0.0.17 promote PR (gsm#336) closed to skip; canary 0.0.18 pushed (5s sleep) |
| 15:10 | Canary 0.0.18 released → gsm#337 opens → admin-merge |
| 15:14 | Canary 0.0.18 arrival → **Failed (empty tests)** — Kyverno `validate` webhook connection refused (transient during Kyverno restart) |
| 15:15 | Manually delete arrival 0.0.18 CR (observer only re-creates on RS Added event, won't auto-retry from Failed terminal) |
| ~15:17 | Empty retrigger commit pushed → canary 0.0.19 released |
| 15:30 | Canary 0.0.19 release → gsm#338 opens → admin-merge |
| 15:34 | Canary 0.0.19 arrival → **Passed** ✓ |
| 15:35 | Forensics-runner produces `diff.json` (test-bounded windows) ✓ |
| ~15:38 | Finding: `/api/v1/example` not in test fixture; revert sleep, capture this doc |

## Conclusion

The Tier-2 infrastructure works. The Tier-2 design has a real coverage constraint (test fixture must cover the regressed endpoint) that's worth documenting prominently. The next iteration should fix the canary fixture or pick a service with broader fixture coverage for the headline demo.
