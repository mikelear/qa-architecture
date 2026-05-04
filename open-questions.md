# Open questions

Decisions still pending or worth re-evaluating before / during build. Add new questions as they surface; move resolved ones into the relevant component doc with a citation.

## Result store

### Q1. GCS, Mongo, or GitHub artifacts? — RESOLVED 2026-04-30

**Decision**: GCS, specifically the existing `gs://test-artifacts-product-first/` bucket extended with a `results/v1/` prefix for SHA-keyed verdicts (alongside the existing PR-keyed artifact uploads).

**Why confirmed**: The bucket + auth pattern (`test-artifacts-gcs-key` ExternalSecret) are already in production via `~/leartech/leartech-pipeline-catalog/tasks/end2end-ui/pullrequest.yaml`. Extending the existing infra costs ~2 hours; building parallel Mongo / GitHub-artifacts infra would cost days. Compromise still applies: GCS as source of truth + thin Mongo indexer for dashboards if/when needed (deferred until needed).

**Migration path note**: existing artifact-upload paths (`<repo>/pr-<N>/<cluster>/<ts>/...`) stay PR-keyed; new results JSONs go to SHA-keyed `results/v1/<repo>/<sha>/<cluster>/<test-pack>/<test-name>.json`. Results JSONs reference artifact URLs for traceability.

### Q2. Result schema versioning

**Proposal**: `schema_version: v1` field; bump on incompatible changes; gate ignores results with unknown versions (logs warning).

**Decision needed**: confirm; alternative is unversioned + guarantee backward-compatibility forever (more constraining).

### Q3. Cross-cluster result merging

`gs://leartech-qa-results/v1/<repo>/<sha>/<cluster>/<test-pack>/<test-name>.json` — gate looks up both clusters separately and applies a policy.

**Decision needed**: are there tests that should only run on one cluster? If so, how is that declared in `required-tests/<service>.yaml`?

## QA management repo

### Q4. CUE vs JSON Schema for validation

**Proposal**: CUE — stricter, more expressive, growing tooling.

- **CUE**: stricter type system, can encode constraints (e.g. `flake_budget: >0 & <1`), tooling improving but still niche.
- **JSON Schema**: ubiquitous tooling, weaker constraint language, more verbose.

**Decision needed**: leans CUE for stricter validation but team familiarity may favor JSON Schema. Worth a 1-day spike comparing the actual schemas.

### Q5. Helm chart vs. raw repo for consumer publishing

If qa-management is published as a Helm chart, consumers can pin via `chart-version: v1.42.0` exactly like other Helm deps. Renovate already understands this.

If raw repo, consumers fetch via init container with a Git tag (mqube's pattern).

**Decision needed**: Helm chart for the parts consumed by services in cluster (test-suite definitions, load configs); raw repo for the parts consumed by leartech-gate (built into the gate image at build time). Hybrid.

### Q6. Override audit log update mechanism

`gate-metadata/overrides.log.jsonl` — how is it appended to?

- Lighthouse webhook plugin → posts to a small webhook receiver service → opens a one-line PR to qa-management
- Or: scheduled scan of Lighthouse audit logs → batched daily PR
- Or: direct write to GCS (not the YAML), backed by a viewer that joins on demand

**Decision needed**: lean toward scheduled scan (low coupling, low operational burden). Direct writes to a YAML log file are append-correct but contended.

## PR-time companion PRs

### Q7. Threshold for opening companion PR

Don't want a companion PR for every PR. Threshold proposed: companion only opens if AI scanner finds `added > 0` actual proposals (i.e. the PR introduces new test surface that's not already covered).

**Decision needed**: confirm; alternative is per-PR companion that's mostly empty (cleaner audit but more noise).

### Q8. `coverage-deferred` TTL

Default 30 days proposed. Re-block after expiry.

**Decision needed**: longer (60-90 days) for some categories of work? Per-repo override?

### Q9. Bulk repo-type.yaml seed strategy

Three options:
- AI-suggested via classifier (per-repo PRs); CODEOWNERS confirm
- Manual seed by repo owners
- Default-by-detection (file sniffing); manual override

**Decision needed**: lean AI-suggested + CODEOWNERS-confirm. Cheapest correctness/effort tradeoff.

## Gate

### Q10. Quill blocking-vs-alert default

`load-sla` proposed as alert-only initially (until baseline is collected). All others blocking from day one.

**Decision needed**: confirm; some teams may want a "warm-up period" for newly-added quills before they block.

### Q11. Approver-required overrides

Proposed: blocking-quill overrides require a separate `/approve override` from a different actor (two-key).

**Decision needed**: confirm; can add cost but reduces single-actor risk. May be too heavyweight for early days.

## HAR pipeline

### Q12. Body-stripping policy

Keep request bodies in HAR (more useful for replay) or strip (safer, less powerful)?

- **Keep**: needed for POST/PUT replay to be meaningful. PII risk if sanitize misses something.
- **Strip**: safer; replay limited to GET/DELETE; POST bodies must come from OpenAPI synthesis.

**Decision needed**: lean keep + invest in robust sanitize (+ maybe per-service body-stripping override for high-PII services).

### Q13. HAR retention

90 days default proposed. Compliance review needed: any auth/audit logs in HAR (even after sanitize) that have a different retention obligation?

**Decision needed**: confirm with security/compliance.

### Q14. Producer priority order

Phase 1 = Playwright. Phase 2 = Tempo synth. Order of additional producers (Phase 2-3):

- Gateway log → HAR
- OpenAPI synth → HAR
- Pixie / Hubble

**Decision needed**: depends on what's available — if leartech has a gateway with structured logs, gateway-log is cheap; if not, OpenAPI synth is the next step.

## Renovate

### Q15. Load-bearing dep list

Initial list proposed (auth libs, framework majors, DB drivers, web frameworks). Should be curated, not just intuition.

**Decision needed**: security-team review of the list before bulk-applying.

### Q16. Patch-version skip policy

Skip full-QA on patch updates? Risks letting patched-but-still-regressing CVEs through.

**Decision needed**: probably skip for true patches (patch-only diff, no API changes); run full-QA if Renovate detects any breaking change in the dep's changelog.

## Multi-cluster

### Q17. Slack-alert layer scope

Phase 3 conditional. If built:

- Per-failed-quill alerts on the source-PR repo's Slack channel
- Daily / weekly digest in a #releases channel
- Per-actor digest (e.g. high override rate by one person → DM)

**Decision needed**: defer to Phase 3 design pass.

### Q18. Tenant model

If leartech grows multi-tenant (à la mqube's nbs-uat / monbs-uat / tresta-uat), the gate model needs to extend. Currently single-tenant; design assumes that.

**Decision needed**: not now; flag if/when tenancy emerges.

## Foundation leverage tracking

### Q19. Where to register patterns

Proposed: `~/leartech/hub/shared-rules/platform-leverage.md` — list of "pattern X landed; reusable for Y; marginal cost of next instance Z."

**Decision needed**: confirm + when to start (probably when Phase 1 lands, capture the patterns from that build).

### Q20. Estimate calibration

After Phase 1, retrospect against estimates here. If actual was off by >50%, recalibrate Phase 2 estimates.

**Decision needed**: schedule a retro at end of Phase 1.

## Risk-assessor (Phase 2.5)

### Q21. Service ownership map bootstrap

`service-catalog.yaml` covers every service in the leartech estate — needs to be populated.

- **AI-suggested + CODEOWNERS-confirm**: classifier reads each repo's `repo-type.yaml` + observed Tempo edges, proposes catalog entry, opens PR per service. CODEOWNERS confirm.
- **Manual seed**: each service owner contributes their own entry.
- **Hybrid**: AI seeds the structure; service owners refine criticality + consumer/consumed-by per service.

**Decision needed**: lean hybrid (AI structural + human refinement); cheapest correct outcome.

### Q22. Confidence-vs-risk trade-off

When `trace_signal.confidence: low` (preview env minimal, Tempo unavailable), should the classifier default to medium or high risk?

- **Default medium**: less friction; over-flagging acceptable for low-coverage envs
- **Default high**: maximum safety; gate stiffens override approval; may feel heavy-handed for low-impact PRs

**Decision needed**: lean medium for first version (less developer friction; tune up if false-negatives surface).

### Q23. Cross-cluster risk

Does a PR's risk vary by cluster (gcp vs az), or is it cluster-agnostic?

- Cluster-agnostic: simpler; risk is intrinsic to the change, not where it deploys
- Per-cluster: catches cluster-specific issues (e.g. PR that's safe on gcp but breaks az due to env config)

**Decision needed**: lean cluster-agnostic v1; per-cluster as future refinement if cluster-specific patterns emerge.

### Q24. Risk-config governance

`risk-config.yaml` (factor weights, thresholds, bucket boundaries) — who tunes? How often?

- Platform team owns; tuned via PR with metric-based justification
- Quarterly tuning cadence with retrospective on previous period's risk-classification accuracy
- Same review process as `repo-type-policy/test-packs.yaml`

**Decision needed**: platform team owns; quarterly retrospective. Confirm with the team.

## Arrivals-observer (Phase 2.7)

### Q-AO1. Mongo or GCS for arrival docs?

**Proposal**: GCS (consistency with result-store).

- **GCS**: same bucket pattern as result-store; cheap; lifecycle policy auto-expires; reuses Go writer helper
- **Mongo**: matches mqube exactly; richer query for dashboards

**Decision needed**: lean GCS; switch to Mongo only if dashboards demand it (then add a thin indexer like result-store's).

### Q-AO2. Forensics artifact retention

**Proposal**: 90 days like result-store; longer for "interesting" regressions (e.g. ones that led to an incident).

**Decision needed**: confirm; security/compliance review may want longer for audit trail.

### Q-AO3. When to flip `post-deploy-tests` quill from alert-only to blocking?

**Proposal**: per-service flip after 6-8 weeks of stable operation; metric-based (override rate <X%, false-positive rate <Y%).

**Decision needed**: define X, Y thresholds. Initial guess: override-rate <15%, false-positive rate <10%.

### Q-AO4. Production observation policy — when, what tests are safe?

Defer to Phase 3 design pass. Initial outline:

- Read-only / idempotent tests only
- Test data quarantine (no creation, or auto-cleanup)
- PII handling per data-handling policy
- Rate limits on test invocation
- Auth scoping with tight credentials
- Specific service-by-service approval

**Decision needed**: separate design doc in Phase 3 if/when triggered.

### Q-AO5. Should arrivals-observer emit regression-log as a structured PR to qa-management?

**Proposal**: yes — auto-merge for `add-only` (matches coverage-scanner pattern); manual review for any policy change.

**Decision needed**: confirm + format for the regression-log entries.

### Q-AO6. Production-coverage policy if/when expanding

When extending arrivals-observer to production: which test packs run vs which are staging-only? Some Playwright tests are inherently staging (test-data assumptions, broker/admin flows that mutate state) and unsafe in prod.

**Decision needed**: gating mechanism in `required-tests/<service>.yaml` — per-test flag like `prod_safe: true` or `envs: [staging]`.

## Resolution log

| Q# | Decision | Date | Where it lives now |
|---|---|---|---|
| Q1 | GCS — existing `test-artifacts-product-first` bucket extended with `results/v1/` prefix | 2026-04-30 | `gate.md` "Result store contract" + this file under Q1 |
| Q-FC | Fat Controller IS load-bearing for the gate (was wrong in earlier framing) | 2026-05-04 | `~/leartech/Qa-Analysis/findings/02-fat-controller.md` "CORRECTION" + `arrivals-observer.md` |
