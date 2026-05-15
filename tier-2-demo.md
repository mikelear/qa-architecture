# Tier-2 forensics demo — 2026-05-14

## TL;DR

**Tier-2 demo SUCCESS — Layer 1 caught the regression in a PR verdict comment.** Three demo cycles produced six engineering findings — four already addressed (three mid-cycle), two carried forward as Phase 3 follow-up work. The rolling-update transitional artifact (the headline architectural fix) is eliminated. Per-test duration quill (Layer 1) and per-endpoint Tempo span-diff (Layer 2) both deployed + enabled cluster-wide. The captured verdict from `gsm#347` shows Layer 1 correctly flagging `canary 0.0.26`'s smoke `765ms → 5652ms (7.4× / +4887ms)` with linked artifact URLs.

## Captured verdict (the headline demo artifact)

PR `mikelear/jx-build-cluster-gsm#347` (dummy demo trigger) — gate-cli sticky comment for canary entry:

```
| leartech-qa-canary | 0.0.26 | ❌ | post-deploy: Arrival.phase=Passed; layer1: 1 duration regression(s) vs 0.0.23 |
|  |  |  | Failed: end2end/smoke: 765ms → 5652ms (7.4× / +4887ms) |
|  |  |  | end2end artifacts: [HTML report] · [trace.zip listing] |
```

Other 18 services in the helmfile all evaluated gracefully (pass-through with explanatory reasons like "no current arrival" / "no previous arrival for baseline" / "Arrival.phase=Skipped (no testPacks)"). This validates the multi-service, mixed-state evaluation path end-to-end.

Pipelinerun: `build-cluster-gsm-pr-347-qa-gate-hbrwn` — pipeline `fail` (correct: canary regression flagged), other services correctly pass-through.

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

### Finding #5 — Tempo endpoint maps empty on 0.0.23 **[RESOLVED 2026-05-15]**

`diff.json` for 0.0.23 (post-rollout-gate) showed both endpoint maps empty. Smoke ran successfully (results.json proved it) but `/api/v1/example` traces never appeared in Tempo within the test window.

**Diagnosis 2026-05-15**:

After bypassing forensics-runner and querying Tempo directly via TraceQL (`{name=~".*example.*"}`), the smoking gun: spans for `/api/v1/example` exist in Tempo for **external** curl probes (`9` matches in the last hour from my laptop), but **zero** matches across the 0.0.21/22/23/26/27 arrival windows. Same handler, same instrumentation.

Crucially: canary pod `6vpxt` started at `17:31:23Z` — only **11 seconds** before the 0.0.27 arrival window began. Pattern:

| Source | Pod age | Result |
|---|---|---|
| External curl on long-running pod | hours | spans appear within seconds |
| Smoke from Job pod, freshly-deployed canary | <1 minute | zero spans (the bug) |
| Kubelet liveness probes (constant) | always present | spans appear (these are what we'd been seeing) |

**Root cause**: cold-start race between pod startup and OTel `BatchSpanProcessor` (5-second batch timeout default). `/health/*` spans always appear because kubelet probes them every 10s, keeping the batch full enough to flush. `/api/v1/example` is called ONCE during smoke; single span queued, BatchSpanProcessor 5s timer doesn't flush before the test window closes + the gate queries Tempo. Span eventually exports but lands too late.

**Fix (PR `leartech-qa-canary#24`, 2026-05-15)**: two-line change to `internal/tracing/tracing.go`:

A. `WithBatchTimeout(1*time.Second)` (was 5s) — shrinks the cold-start race window
B. Pre-warm the exporter: emit + `ForceFlush` a single dummy span BEFORE the HTTP server starts accepting traffic. Forces DNS + TCP + HTTP/2 handshake + first OTLP accept during startup, not racing the first real request.

**Validation (canary 0.0.28 arrival, 2026-05-15 08:51-08:52Z)**:

```
after endpoints: ['/api/v1/example', '/docs', '/health/live', '/health/ready', '/openapi.json']
/api/v1/example p95: 0.18961ms
```

For the first time across 7 arrivals (0.0.21/22/23/26/27/28), `/api/v1/example` is present in the diff.json `after` endpoint map. The two-line fix worked.

**Phase 2 follow-up**: extract this pattern to `leartech-go-common/pkg/tracing` (today: `auth/httptools/lock/logger/mongo/queue/redis` — no `tracing`). Update `arrivals-observer/internal/tracing/tracing.go` + `forensics-runner/internal/tracing/tracing.go` to use it. Add tracing to services that lack it (`auth-service/auth-ui/ai-classifier/gate` — all currently producing zero spans in Tempo).

**Rejected alternatives** (per research agent):
- `SimpleSpanProcessor` — per-span synchronous export under mutex; problematic at any meaningful QPS
- `X-Smoke-Test` header that triggers ForceFlush — couples service code to test infra; A+B was sufficient

### Finding #6 — Layer 1 AND-gate too conservative for high-overhead test fixtures **[UPDATED in Cycle 3]**

Originally raised when baseline was 6821ms (pre-rollout-gate):

- `+5000ms` delta on `/api/v1/example` → smoke 6821 → ~11800 ms
- Ratio = `11800/6821 = 1.73×` ← **below** the 3× threshold

Layer 1's AND-gate (ratio ≥3× AND delta ≥500ms) was designed against small-duration test fixtures (where 60→200ms is 3.3× / +140ms = noise). On a 7-second high-overhead fixture, a 5s real regression falls **inside** the noise threshold.

**Cycle 3 update:** the rollout-gate's baseline shift (Finding #4 → smoke now 765ms, not 6821ms) **inadvertently resolved this for canary**. With 765ms baseline + 5s sleep:
- smoke 765 → 5652ms = **7.4× ratio + 4887ms delta** → both AND-gate thresholds exceeded → **flagged cleanly**

This means the AND-gate is well-tuned for *small-baseline* fixtures (which is what canary now is post-rollout-gate). For *real production services* with multi-second smoke baselines (e.g. an auth flow that does login + redirect + assertion = 3-4s), the AND-gate may still over-filter. Worth investigating per-service threshold customisation or proportional sensitivity in Phase 3 — but no longer blocks the demo.

Phase 3 design options (deferred):
- Per-service threshold config via qa-management (mirrors mqube's required-tests pattern but for sensitivity tuning)
- Proportional sensitivity: `max(ratio, delta_pct_of_baseline) ≥ 30%`
- Per-endpoint Layer 1 detection (would need `results.json` schema extension to emit per-endpoint timings within a test)

### Finding #7 — jx-verify GC churn causes promote PR conflicts (operational)

`gsm#344` (canary 0.0.24 promote) hit a merge conflict on 10 files:
- 4× `jx-verify/jx-verify-gc-jobs-*-{job,rb}.yaml` (auto-renamed periodically)
- 6× canary-related config-root files (rewritten by helm template)

The jx-verify GC job filenames churn frequently (every few minutes during active CI). Promote PRs opened by updatebot at one point in time conflict against later-renamed files. The actual chart-version-bump intent is unambiguous, but git merge can't resolve.

Mitigation options (low priority — operationally noisy but not breaking):
- Rebase auto-promote PR branches on top of main when they age (updatebot lacks this today)
- Move jx-verify GC files out of config-root (declarative regen)
- `auto-merge` on promote PRs so they land before the next GC rotation

## Cycle 3 — 2026-05-14, ~17:00-17:20 UTC — **option A close-out (SUCCESS)**

### Goal

Re-run Phase 2 with canary 0.0.25 (clean rollout-gated baseline) → canary 0.0.26 (5s sleep) so the smoke baseline is small enough for the AND-gate to trigger on the regression. Then capture the gate verdict comment.

### What landed

| Time (UTC) | Event |
|---|---|
| 17:01 | canary 0.0.26-gcp released (sleep re-injected via 507392c) |
| 17:02 | gsm#346 (canary 0.0.26 promote) opened |
| 17:02 | gsm#346 auto-merged (no jx-verify GC conflict this time) |
| 17:10 | canary 0.0.26 arrival landed (Passed) — observer's rollout-gate worked: tests ran AFTER new pod was the only one serving |
| 17:12 | forensics-runner ran (Tempo still produced empty endpoint maps — Finding #5 persistent) |
| 17:13 | gsm#347 (dummy demo trigger) opened to provoke gate-cli evaluation |
| 17:18 | gate-cli ran on gsm#347 — verdict comment posted |

### Captured Layer 1 verdict (see TL;DR)

Layer 1 cleanly detected the regression and posted the verdict-comment with artifact links. Pipelinerun `build-cluster-gsm-pr-347-qa-gate-hbrwn` exit code: fail (correct — canary's entry flagged, gate check goes red, branch protection would block the merge if this were a real promotion PR).

### What Cycle 3 validates that Cycle 1+2 didn't

| Component | Cycle 3 result |
|---|---|
| Layer 1 verdict comment renders in PR | ✅ |
| Layer 1's AND-gate correctly fires on a real regression | ✅ (7.4× / +4887ms) |
| Artifact URLs (HTML report + trace listing) render in verdict | ✅ |
| Mixed-state evaluation across 19 services (some deployed, some not, some skipped) | ✅ — all 18 non-canary services pass-through gracefully with explanatory reasons |
| gate-cli's exit code drives the GitHub check status correctly | ✅ (`gcp/qa-gate fail`) |
| `/override leartech-gate` instructions surface in the comment | ✅ |

### Finding #5 (Tempo empty endpoint maps) — PERSISTED ON 0.0.26

Confirmed: `diff.json` for 0.0.26 shows zero latency_regressions, `null` for both `before /api/v1/example` and `after /api/v1/example` despite the 5s sleep clearly executing (results.json proves smoke ran). This is now confirmed-persistent across multiple arrivals post-rollout-gate. Phase 3 follow-up item #1.

The verdict still shows post-deploy's forensics summary slot (no latency_regressions reported), but Layer 1's per-test signal carries the demo. This is exactly the two-layer design intent — when one layer is unavailable/unhelpful, the other still produces actionable output.

## What the three cycles together prove

| Component | Status |
|---|---|
| Observer fires forensics on Passed (PR #6) | ✅ verified in `Arrival.status.forensics.generatedAt` on canary 0.0.19/0.0.21/0.0.22/0.0.23/0.0.26 |
| Forensics test-bounded windows (PR #6) | ✅ `before_window` + `after_window` correctly derived from CR |
| Observer rollout-wait gate (PR #7) | ✅ deployed (observer 0.0.24); eliminated the rolling-update transitional artifact |
| Forensics handles "no previous arrival" (first deploy) | ✅ exercised inadvertently on early arrivals |
| diff.json upload to GCS at controller-rendered path | ✅ landing at expected `forensics/v1/gcp/jx-staging/leartech-qa-canary/<ver>/diff.json` |
| `Arrival.status.forensics.diffUrl` patched on every arrival | ✅ |
| Layer 1 deployed + flag-enabled | ✅ Cycle 3 fired correctly (7.4× regression flagged) |
| **Layer 1 verdict-comment-in-PR with artifact links** | ✅ **Cycle 3 captured (gsm#347)** |
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

## Phase 2 close-out — 2026-05-15 (post-cycle 3)

After Tier-2 Phase 1 validated the tracer fix on canary 0.0.28 (`/api/v1/example` p95=0.19ms in `after endpoints` for the first time), Phase 2 extracted the pattern to `leartech-go-common/pkg/tracing` and propagated to all consumers.

| PR | Repo | Status | What |
|---|---|---|---|
| #3 | `leartech-go-common` | ✅ merged | New `pkg/tracing` with 5 tests, cold-start race fixes baked into defaults |
| #27 | `leartech-qa-canary` | ✅ merged | Local `internal/tracing` → `go-common/pkg/tracing` (validated on canary 0.0.30 — `/api/v1/example` p95=0.24ms preserved) |
| #8 | `leartech-arrivals-observer` | ✅ merged | Same swap; observer added `leartech-go-common` as new dep |
| #7 | `leartech-forensics-runner` | ✅ merged | Deleted unused local tracing pkg (dead code from bootstrap, never wired into main.go) |

Adjacent gate-discipline shipped during the same cycle:

| PR | Repo | What |
|---|---|---|
| #31 | `leartech-pipeline-catalog` | Delta-coverage gate v1 (PR must not drop coverage > 0.5% vs base) |
| #32 | `leartech-pipeline-catalog` | Fresh-clone base for delta-coverage (was leaking PR's untracked files) |
| #33 | `leartech-pipeline-catalog` | mktemp portability fix (GNU vs BSD `-t` flag) |
| #25 | `leartech-qa-canary` | Unit tests for canary tracing (closes PR#24 coverage gap) |

### Phase 3 — open work (separate sessions)

- **#129**: extract tracing to `leartech-go-common/pkg/tracing` — done above
- **#130**: `leartech-go-common` test backfill across 8 existing pkgs (auth/httptools/lock/logger/mongo/queue/redis) — mirror mqube's `pkg/testutils` pattern. Currently 0 tests; floor at 5% with explicit ratchet plan to 60%
- **#131**: clean up 14 pre-existing lint issues in `leartech-go-common` (7 are API-breaking naming-stutter renames needing coordinated consumer updates)
- **#132**: `leartech-go-common/pkg/auth/config.go:24` hardcoded cluster URL — semgrep flagged
- **#133**: counter-test PR to validate delta-coverage gate's FAIL path (gate's PASS path validated on PR#3 + PR#25, but a deliberate-regression PR hasn't been engineered yet)
- **Phase 3 tracing**: add tracing to services that have none (auth-service, auth-ui, ai-classifier, gate — all currently produce zero Tempo spans)
