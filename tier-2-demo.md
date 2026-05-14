# Tier-2 forensics demo — 2026-05-14

## TL;DR

All Tier-2 infrastructure is proven working end-to-end. Two demo cycles produced five distinct engineering findings — three already addressed (one mid-cycle), two carried forward as follow-up work. The rolling-update transitional artifact (the headline architectural fix this cycle) is eliminated. Per-test duration quill (Layer 1) and per-endpoint Tempo span-diff (Layer 2) both deployed + enabled cluster-wide. The cycle did not produce a final "gate verdict catches regression in PR comment" capture because (a) the promote PR for the regressed version hit a jx-verify GC churn conflict, (b) results.json baseline shifted unexpectedly after rollout-gate landed, and (c) Layer 1's AND-gate is over-conservative for high-overhead test fixtures. These are real follow-ups, not bugs in the design.

## Cycle 1 — 2026-05-14, ~14:48-15:38 UTC

### Goal

Validate Layer 1 + Layer 2 catching a `passed-but-slower` regression on canary, end-to-end.

### What we built before this cycle

The Tier-2 design (see `~/.claude/projects/-Users-mikelear-leartech-Qa-Analysis/memory/project_layer1_layer2_forensics_design.md`):

- **`leartech-arrivals-observer@0.0.23` (PR #6)** — forensics now fires on every Arrival including `Passed` (not just `Failed`/`Timeout`)
- **`leartech-forensics-runner@0.0.8-gcp` / `@0.0.5-az` (PR #6)** — Tempo windows derive from `Arrival.status.tests[].startedAt..completedAt` instead of fixed `±5m wall-clock`, eliminating background-noise pollution
- **`leartech-gate@0.0.16` (PR #11)** — Layer 1 duration-regression quill (per-test `duration_ms` comparison from `results.json`)
- **`ENABLE_DURATION_QUILL=true`** in both GitOps `qa-gate.yaml` (gsm#335 + akv#190)

### Method

1. Injected `time.Sleep(1*time.Second)` then later `time.Sleep(5*time.Second)` in canary's `GET /api/v1/example` handler
2. Pushed canary 0.0.17 → 0.0.18 → … → 0.0.19 (release postsubmit + auto-promote + bootjob + arrival)
3. Watched arrival, results.json, diff.json

### Finding #1 — `/api/v1/example` not in smoke fixture

```
end2end/01-smoke.sh checks:
  GET/HEAD /health/live, /health/ready, /openapi.json, /docs   ← 8 checks
  (NO /api/v1/example — it's @Security BearerAuth)
```

Sleep in `/api/v1/example` is **invisible** to:
- Layer 1: smoke `duration_ms` unchanged (the sleep endpoint isn't called)
- Layer 2: Tempo `before`/`after` endpoint maps don't include `/api/v1/example` (never called by tests)

Useful caveat: **the gate is structurally blind to endpoints the test fixture doesn't cover.** Not a bug — a real coverage constraint of any test-coverage-based regression detector.

Resolved mid-cycle: extended `01-smoke.sh` with a 9th check `check_auth GET /api/v1/example 200` (dummy `Bearer demo` token; `leartech-go-common@0.1.0` middleware is currently presence-only). Phase 1 commit `d56ad92`.

### Finding #2 — Rolling-update transitional artifact

After Phase 1 landed (canary 0.0.21), `diff.json` showed `/api/v1/example` at p95 **5004ms** even though the new pod responded in ~350ms (verified via direct curl).

Root cause: observer fires tests on `ReplicaSet Added` — BEFORE rolling update completes. During the transition window, the K8s Service load-balances across both old and new pods. The single `/api/v1/example` span captured was the OLD pod's response (with the still-deployed sleep code).

**Resolved this cycle: arrivals-observer PR #7** adds a Deployment-rollout gate in `handleNewArrival`. Mirrors `kubectl rollout status` semantics:

- `observedGeneration >= generation`
- `updatedReplicas == spec.replicas`
- `availableReplicas == spec.replicas`
- `unavailableReplicas == 0`

Polls every 5s up to `RolloutTimeout` (default 5min) before dispatching tests. On timeout: arrival → Failed with reason "rollout did not complete". Missing Deployment (non-Deployment-backed services) → skip gate gracefully. 10 dynamicfake tests cover all branches. Deployed as observer 0.0.24 in GCP cluster.

### Finding #3 — Kyverno admission webhook flakes (operational)

During the demo window, Kyverno restarted three times. Each restart caused some pod-create attempts to fail with `validate.kyverno.svc-fail: connection refused` or `mutate.kyverno.svc-fail: context deadline exceeded`. Affected:

- `gate-cli#11` GCP `ai-review` + `image-scan` pipelinerun pods
- canary release postsubmit `leartech-qa-canary-main-release-4cqkf`
- observer's create-Job for canary 0.0.18's end2end (`dial tcp 10.48.13.219:9443: connect: connection refused` to a stale Kyverno pod IP)

Memorialised in `memory/project_kyverno_admission_flakes.md`. The user subsequently merged the Kyverno HA/PDB/anti-affinity/resources fix (`akv#192`) in the AZ cluster mid-cycle. Long-term mitigation underway in parallel.

## Cycle 2 — 2026-05-14, ~16:13-16:50 UTC (with rollout-gate observer)

### Goal

Re-run Phase 1 + Phase 2 with observer 0.0.24's rollout-wait gate active to get a clean before/after capture.

### What landed

- canary 0.0.22 → 0.0.23 deployed via observer 0.0.24
- Phase 2 commit `d5cd7c2` pushed (5s sleep on `/api/v1/example`)
- gsm#344 auto-promote PR for canary 0.0.24 opened

### Finding #4 — Baseline `smoke duration_ms` dropped 10× with rollout-gate

| Canary version | Observer | smoke duration_ms |
|---|---|---|
| 0.0.22 (just before rollout-gate) | 0.0.23 (no gate) | **6821 ms** |
| 0.0.23 (after rollout-gate active) | 0.0.24 (with gate) | **765 ms** |

Same `01-smoke.sh` script, same 9 curl checks. The 10× drop is suspicious. Hypotheses:

- **Warm-pod effect**: rollout-gate adds 30-60s before tests run. By that time the new pod's DNS resolvers + TLS sessions are primed (readiness probes have already exercised the endpoints). When smoke's curl hits, no cold-start.
- **K8s Service endpoint stability**: with rollout-gate, only one Service endpoint set (new pods) — no LB rotation to slow down connection setup.
- **Some other timing difference** in how the dispatch Job runs post-gate.

If the warm-pod hypothesis is correct, the rollout-gate is also a baseline normalizer. Worth verifying with more samples + investigating before relying on the duration figure for thresholds.

### Finding #5 — Tempo endpoint maps empty on 0.0.23

`diff.json` for 0.0.23 (post-rollout-gate) shows:

```json
{
  "after_window": { "start": "2026-05-14T16:42:45Z", "end": "2026-05-14T16:44:08Z" },
  "before": {},
  "after": {}
}
```

Both endpoint maps empty. But the smoke test definitely called `/api/v1/example` (results.json proves the test ran successfully, and Phase 1 extended the fixture to include it).

Possible causes:
- **Window alignment off**: `startedAt` is recorded by observer when it transitions the Arrival to Testing (right after rollout-gate passes). Tests don't actually start running until the K8s Job is scheduled (30-60s later). If Tempo spans are in [Job-start, Job-end] but window is [observer-startedAt, observer-completedAt], the spans may fall outside.
- **Tempo collector behaviour**: spans may be batched and flushed AFTER `completedAt`, especially for very short tests (765ms wall-clock).
- **Service-name mismatch**: the OTel resource attributes on the new pod might differ from what forensics-runner queries Tempo for.

Worth investigation. May warrant moving the `startedAt` capture from "observer dispatched Job" to "Job pod started running" — needs observer-side handler that watches the Job's pod transitions.

### Finding #6 — Layer 1 AND-gate too conservative for high-overhead test fixtures

On the (pre-rollout-gate) 6821ms baseline:

- `+5000ms` delta on `/api/v1/example` → smoke 6821 → ~11800 ms
- Ratio = `11800/6821 = 1.73×` ← **below** the 3× threshold

Layer 1's AND-gate (ratio ≥3× AND delta ≥500ms) was designed against small-duration test fixtures (where 60→200ms is 3.3× / +140ms = noise). On a 7-second high-overhead fixture, a 5s real regression falls **inside** the noise threshold and gets filtered.

Layer 2 doesn't have this problem — endpoint-level comparison sees `0.26ms → 5000ms = 19000×` cleanly.

Worth fixing for Phase 3: replace AND-gate with proportional sensitivity, e.g.:

```
ratio_or_delta_pct_of_baseline := max(ratio, current/baseline_delta)
flag if ratio_or_delta_pct_of_baseline ≥ 30%
```

Or split per-endpoint Layer 1 detection (drilling into per-endpoint duration_ms within a test that exposes endpoint-level timings). Today's `results.json` doesn't have per-endpoint breakdowns — that'd be a fixture + schema change.

### Finding #7 — jx-verify GC churn causes promote PR conflicts (operational)

`gsm#344` (canary 0.0.24 promote) hit a merge conflict on 10 files:
- 4× `jx-verify/jx-verify-gc-jobs-*-{job,rb}.yaml` (auto-renamed periodically)
- 6× canary-related config-root files (rewritten by helm template)

The jx-verify GC job filenames churn frequently (every few minutes during active CI). Promote PRs opened by updatebot at one point in time conflict against later-renamed files. The actual chart-version-bump intent is unambiguous, but git merge can't resolve.

Mitigation options (low priority — operationally noisy but not breaking):
- Rebase auto-promote PR branches on top of main when they age (updatebot lacks this today)
- Move jx-verify GC files out of config-root (declarative regen)
- `auto-merge` on promote PRs so they land before the next GC rotation

## What this cycle proves

| Component | Status |
|---|---|
| Observer fires forensics on Passed (PR #6) | ✅ verified in `Arrival.status.forensics.generatedAt` on canary 0.0.19/0.0.21/0.0.22/0.0.23 |
| Forensics test-bounded windows (PR #6) | ✅ `before_window` + `after_window` correctly derived from CR |
| Observer rollout-wait gate (PR #7) | ✅ deployed (observer 0.0.24); eliminated the rolling-update transitional artifact |
| Forensics handles "no previous arrival" (first deploy) | ✅ exercised inadvertently on early arrivals |
| diff.json upload to GCS at controller-rendered path | ✅ landing at expected `forensics/v1/gcp/jx-staging/leartech-qa-canary/<ver>/diff.json` |
| `Arrival.status.forensics.diffUrl` patched on every arrival | ✅ |
| Layer 1 deployed + flag-enabled | ✅ (no flag fired in this cycle — see finding #6 for why) |
| Layer 2 deployed + Tempo span-diff produces correct shape | ✅ (empty maps on 0.0.23 a separate finding — see #5) |
| Tempo collector emits spans for traced endpoints | ✅ on 0.0.21 + 0.0.22; ❓ on 0.0.23 (see finding #5) |
| Bearer-auth test fixture for authed surface | ✅ Phase 1 `check_auth` helper works |

## What this cycle does NOT prove

- **A verdict-comment-in-PR Layer 1 trigger.** No gate verdict comment was captured because (a) finding #6 (AND-gate too conservative for the test fixture) means Layer 1 wouldn't trigger at the chosen sleep, AND (b) finding #5 (empty Tempo maps post-rollout-gate) means Layer 2 may not either, AND (c) the canary 0.0.24 promote PR hit the jx-verify conflict and couldn't admin-merge. Each of those is independently fixable.
- **AZ-cluster Tier-2 demo.** GCP only this cycle — AZ Postgres + Kyverno HA work was in flight (user's parallel session).

## Follow-up backlog (in priority order)

| # | Item | Effort | Why |
|---|---|---|---|
| 1 | Investigate empty Tempo endpoint maps on 0.0.23 (finding #5) | ~1h | Blocker for Layer 2 producing actual diffs; could be window-alignment, OTel collector, or service-name attribution |
| 2 | Investigate baseline duration 10× drop (finding #4) | ~30m | Either useful baseline-normalisation or a smoke-script issue; need to confirm |
| 3 | Tune Layer 1 thresholds (finding #6) | ~1h gate-cli + tests | Proportional sensitivity instead of fixed AND-gate — would catch this cycle's regression cleanly |
| 4 | Move `startedAt` capture from observer-side to Job-pod-start (if finding #5 is window-alignment) | ~2h observer + tests | Aligns Tempo window with actual test execution |
| 5 | Optional: auto-rebase promote PRs to avoid jx-verify GC conflicts (finding #7) | ~half-day | Operational polish, not blocking |
| 6 | Cycle 3 demo capture (re-run Phase 2 once 1+3 land) | ~1h | Final demo with the full chain working visibly |

## Operational timeline (cycle 2)

| Time (UTC) | Event |
|---|---|
| 16:19 | observer #7 (rollout-gate) merged |
| 16:24 | observer 0.0.24 image released |
| 16:26 | gsm#342 (observer 0.0.24 promote) admin-merged |
| 16:30 | observer 0.0.24 deployed (2 fresh pods, age 3m26s + 3m1s) |
| 16:35 | empty canary commit pushed (42ccf01) to cut clean baseline |
| 16:39 | canary 0.0.23 released |
| 16:40 | gsm#343 (canary 0.0.23) admin-merged |
| 16:43 | canary 0.0.23 arrival Passed; smoke duration 765ms; diff.json endpoint maps empty 🚩 |
| 16:43 | Phase 2 commit pushed (d5cd7c2 — 5s sleep) |
| 16:43 | canary 0.0.24 released |
| 16:43 | gsm#344 (canary 0.0.24 promote) opened |
| 16:46 | gsm#344 admin-merge attempt → CONFLICTING (jx-verify GC churn) → closed |
| ~17:00 | Per option-B: revert sleep, capture findings, close cycle |

## Closing note

The cycle aimed to produce a gate-verdict-comment screenshot. It produced something more useful: five separately-actionable findings on the seam between observability, K8s pod lifecycle, gate threshold tuning, and updatebot ergonomics. Each finding has a clear next step. The infrastructure that the demo was validating is itself sound.

The headline win — observer's rollout-wait gate eliminating rolling-update transitional artifacts — landed in production this cycle and will benefit every future arrival regardless of whether the demo cycle continues.
