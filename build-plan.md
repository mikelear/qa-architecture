# Build plan — phased, foundation-leverage-aware

Three phases. Each phase ships independently and has standalone value. Total effort with foundation leverage: ~5 weeks across 2-3 engineers (or ~10-12 weeks for one engineer single-threaded). Phase 1 is the closed loop; everything else is enrichment.

Estimates assume the existing leartech foundation (`leartech-go-service-template`, `leartech-pipeline-catalog`, `preview-shift-left`, `leartech-ai-classifier`, self-hosted Renovate, multi-cluster JX3, Grafana stack) is reused. **Without** that leverage, each estimate roughly doubles or triples.

## Phase 1 — closed loop (~2 weeks)

The smallest set that delivers an automated QA gate on production-promotion PRs. After Phase 1, every promotion to leartech production is gated on the same shift-left tests that already run pre-merge — automated, with override.

### Deliverables

| Item | Effort | Owner-fit |
|---|---|---|
| `leartech-qa-management` repo: schemas (CUE), `required-tests/`, `repo-type-policy/`, `gate-metadata/quills.yaml` | 3 days | Platform |
| Result store: extend existing `gs://test-artifacts-product-first/` with SHA-keyed `results/v1/` prefix; results-write helper in `leartech-go-common` | **~half day** (bucket + auth already exist via `end2end-ui` task) | Platform |
| Wire shift-left + contract test results to result-store (extend `end2end-ui` task; add equivalent to other test-pack tasks) | 1.5 days | Platform |
| `leartech-gate` Tekton task + Go binary (3 quills: shift-left-tests, contract-tests, co-promotion) | 5 days | Platform |
| Lighthouse trigger + branch-protection wiring on production GitOps repo | 1 day | Platform |
| Rollout: bulk `repo-type.yaml` seed across existing repos (AI-suggested + human-confirm) | 3 days | Platform |
| Documentation + runbook for `/override leartech-gate` | 1 day | Platform |

**Total**: ~15 person-days = ~3 weeks for one engineer; ~1.5-2 weeks parallelized across 2. (Half-day reduction from prior estimate because the GCS bucket + auth pattern is already in production via `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml`.)

### Validation criteria

- [ ] At least one production-promotion PR opens with `leartech-gate` check appearing
- [ ] Result-store contains shift-left + contract results for SHAs of recent merges
- [ ] A test failure in shift-left correctly fails the gate on the next promotion
- [ ] `/override leartech-gate` works and is logged to `gate-metadata/overrides.log.jsonl`
- [ ] `repo-type.yaml` is present in 100% of active service repos

### Phase 1 explicitly does NOT include

- AI-suggested companion PRs (Phase 2)
- HAR pipeline / load testing (Phase 2)
- Renovate hardening (Phase 2)
- Tempo→HAR producer (Phase 2)
- Slack alerting (Phase 3)
- Synthetic prod probes (Phase 3)
- Hubble/Pixie (Phase 3, conditional)

## Phase 2 — enrichment (~2 weeks)

Adds the AI coverage suggester, HAR pipeline, load testing with SLAs, and Renovate hardening. After Phase 2, the system has all the major value from mqube's design plus the structural improvements specific to leartech.

### Deliverables

| Item | Effort | Owner-fit |
|---|---|---|
| AI coverage scanner Tekton task: detects new test surface, opens companion PRs to qa-management | 3 days | Platform + AI |
| Companion PR auto-merge + drift labels (`coverage-deferred` with TTL) | 2 days | Platform |
| HAR pipeline: Playwright HAR storage to GCS + sanitize-on-write (fork mqube's sanitize utility) | 3 days | Platform |
| `leartech-load-testing` Go service (lift mqube-load-testing pattern) | 3 days | Platform |
| `load-sla` quill in leartech-gate (alert-only first, blocking after baseline) | 2 days | Platform |
| Nightly load-test CronJobs (per-service from `load/` configs) | 1 day | Platform |
| Renovate hardening: `full-qa` Lighthouse trigger + load-bearing-dep config | 2 days | Platform + Sec |
| `tempo-to-har` synthesizer Go service | 3 days | Platform |
| Coverage-gap reporter (HAR diff: production vs tested) | 2 days | Platform |
| Documentation + dashboards | 2 days | Platform |

**Total**: ~23 person-days = ~3-4 weeks for one engineer; ~2 weeks parallelized across 2-3.

### Validation criteria

- [ ] AI scanner opens at least one companion PR per week with `added` entries
- [ ] Companion PRs auto-merge on source PR merge for `added`-only diffs
- [ ] HAR pipeline produces at least 1 new HAR per Playwright run, sanitized + stored
- [ ] `leartech-load-testing` runs nightly per-service against current latest HAR
- [ ] `load-sla` quill present in gate output (initially alert-only)
- [ ] Renovate PRs run full Playwright suite via `full-qa` trigger
- [ ] At least one Renovate-load-bearing-dep PR is correctly blocked by `human-review-required`
- [ ] `tempo-to-har` produces at least 100 synthesized HAR/day
- [ ] Coverage-gap report runs weekly and identifies untested edges

## Phase 2.5 — risk-based gating (~3 weeks)

Adds `leartech-risk-assessor` — a Tekton task that classifies PR risk via static analysis (primary) + Tempo + HAR signals (confirming), and feeds the result into the AI code reviewer, the AI coverage scanner, and `leartech-gate`'s override-approval policy. See `risk-assessor.md` for the full design.

Sits between Phase 2 and Phase 3 because it depends on Phase 2's `tempo-to-har` + HAR pipeline + AI coverage scanner, but is too substantive to bundle into Phase 2 line-items.

### Deliverables

| Item | Effort | Owner-fit |
|---|---|---|
| Service ownership map (`service-catalog.yaml` in qa-management) — bootstrap + maintenance | 3 days | Platform + AI |
| AST analysis Tekton task — Go (`go-callvis` wrapper or custom `go/ast` walker) | 4 days | Platform |
| AST analysis Tekton task — TypeScript (`ts-morph` wrapper) | 3 days | Platform |
| Tempo span collector (extends Phase 2's `tempo-to-har`) | 2 days | Platform |
| HAR endpoint extractor (reads Phase 2's HAR pipeline) | 1 day | Platform |
| Risk classifier Go service (rule-based, data-driven thresholds) | 4 days | Platform |
| Result-store write of risk verdict (extends gate's helper) | 2 hours | Platform |
| `risk_modifiers` extension to `repo-type-policy/test-packs.yaml` | half day | Platform |
| AI code-reviewer integration (cite risk + factors in PR comment) | 1 day | Platform + AI |
| Companion-PR risk-derived test additions (extends Phase 2's coverage scanner) | 1 day | Platform + AI |
| Gate override-policy enforcement (read assessment, gate `/override` accordingly) | 1 day | Platform |
| Documentation + runbook + risk-config.yaml seed | 1 day | Platform |

**Total**: ~20 person-days = ~3 weeks for one engineer; ~2 weeks parallelized across 2.

### Validation criteria

- [ ] AST tools produce a non-empty transitive impact set for at least 10 sample PRs
- [ ] Service ownership map covers 100% of active leartech repos (auto-bootstrapped + CODEOWNERS-confirmed)
- [ ] Risk-assessor produces a verdict for every Phase 1+2 service PR (not just promotion PRs)
- [ ] Risk verdicts cite explicit factors + rationale in the AI code-reviewer PR comment
- [ ] Gate elevates override-approval requirements when risk_level=high (verified by sample PRs)
- [ ] False-positive rate on risk classifier <30% in first calibration window (otherwise tune thresholds)

### What Phase 2.5 explicitly does NOT include

- ML-trained risk model (rule-based stays primary; ML deferred to Phase 3 if rule-based proves insufficient)
- Production traffic ingestion (Tempo at staging only; production probes remain Phase 3)
- Automatic merge based on risk score (developers still merge manually; risk informs **what's required**, not **whether to merge**)

## Phase 3 — strategic / conditional (~variable, only if needed)

Adds richer traffic mapping (Pixie or Cilium+Hubble), Slack alerting, synthetic prod probes, future HAR consumers. **Each item is independent** — pull in only the ones that solve a real pain.

### Conditional deliverables

| Item | Trigger to build | Effort |
|---|---|---|
| Pixie deploy + `pixie-to-har` exporter | Tempo coverage has measurable gaps | 1-2 weeks |
| Cilium CNI migration + Hubble exporter | Other reasons to swap CNI (network policy, mTLS) | months (treat as separate project) |
| Slack-alert layer (mqube Fat Controller equivalent) | Team explicitly wants Slack signal in addition to PR check | 3 days |
| Synthetic prod probes (Tekton CronJob hitting prod endpoints) | First production-only regression slips through Phase 1+2 | 1 week |
| Test-generation AI (HAR diff → propose Playwright tests) | Coverage-gap report shows persistent untested edges | 3 days |
| Contract-derivation tool (HAR → Pact contracts) | Team adopts Pact-style contract testing | 1 week |
| Security-replay (HAR + mutations) | DAST/security team wants automated fuzzing | 1-2 weeks |
| Cronjob health quill (mqube's third quill) | Leartech grows cronjob-heavy services | 3 days |
| Tenant gate (if leartech grows tenant orgs) | Multi-tenant deploy model emerges | 1-2 weeks |

## Sequencing logic

**Why Phase 1 is what it is**: the closed loop is the highest-value smallest unit. Without it, you have shift-left signal but no automated gate. Adding the gate without changing the test runner or HAR pipeline is structurally clean — fewer moving parts to get wrong on the first iteration.

**Why HAR pipeline isn't Phase 1**: it's strategic but not on the critical path for "stop bad releases reaching production." Phase 2 adds it once Phase 1 is stable.

**Why Renovate hardening is Phase 2**: it's small (~2 days) but depends on the `full-qa` Tekton task pattern that emerges naturally from Phase 1's preview-env infrastructure. Land it once that's settled.

**Why Phase 3 is conditional**: each Phase 3 item solves a problem you may not have. Build only when the operational signal says you need it. mqube has Phase 3 items (load testing) but no Phase 1 closed loop with structural improvements — the priority order matters.

## Foundation leverage register

Each phase deliverable is cheaper because of the foundation already in place. Worth tracking the leverage explicitly so future estimates anchor honestly. **Proposed**: `~/leartech/hub/shared-rules/platform-leverage.md` once Phase 1 ships, capturing:

```
Pattern landed: leartech-gate Tekton task with Lighthouse trigger
Reusable for: any future Tekton presubmit on the GitOps repo
Marginal cost of next instance: half day (clone trigger + new task)

Pattern landed: result-store + go-common write helper
Reusable for: any future test/check that needs to record a verdict
Marginal cost of next consumer: 2 hours (just call the helper)

Pattern landed: data-driven quill framework in leartech-gate
Reusable for: new gate checks
Marginal cost per quill: half-to-1 day per quill impl, less for variants
```

This is the artifact that makes future estimates defensible. "How long for the load-sla quill?" → "Half a day, because the framework is settled."

## Risk register

| Risk | Phase | Mitigation |
|---|---|---|
| Shift-left tests are flaky → gate is noisy → constant `/override` → ceremonial | 1 | `flake_budget` + auto-quarantine in `required-tests/<service>.yaml`; weekly override-rate review |
| AI coverage scanner produces too many companion PRs → noise | 2 | Initially gate on per-PR `added: > 5` threshold (don't propose for trivial changes); tune from there |
| HAR pipeline storage cost grows fast | 2 | 90-day lifecycle policy; keep only "interesting" HARs (failures, perf outliers) longer |
| Tempo coverage is poor → tempo-to-har is empty | 2 | Validate during Phase 1 by running a one-off Tempo→HAR query; if coverage is bad, adjust Phase 2 priorities (gateway-log → HAR may be a better second producer) |
| Override rate too high to read | 1 | Weekly Slack digest of override stats — exposes the operational health |
| `repo-type.yaml` bulk seed misclassifies repos | 1 | AI-suggest, human-confirm via CODEOWNERS PR review; correctness over coverage |

## Success metrics (post Phase 2)

- **Override rate** on `leartech-gate`: <20% of production-promotion PRs (target). Higher means quill noise; lower means gate is too lenient.
- **Test-list drift**: zero events of "test ran but isn't in qa-management" or vice versa, caught by CI cross-reference linter.
- **Coverage gap closure**: weekly coverage-gap report shows decreasing untested-edge count over time.
- **Renovate-introduced regressions in prod**: zero (Phase 2 should catch them all pre-merge).
- **Time to add a new quill**: <1 day.
- **Time to add a new test pack to the policy**: <half day.

## What to do first thing tomorrow

If you're starting Phase 1:

1. Create `leartech-qa-management` repo with golden-template seed structure
2. Write the CUE schemas (start with `required-tests/<service>.yaml`)
3. Stand up the GCS result-store bucket + IAM
4. Then `leartech-gate` from the Go service template

The gate-itself is the longest critical-path item; everything else can run in parallel once the schemas are settled.
