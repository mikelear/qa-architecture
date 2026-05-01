# Renovate hardening — full QA on dependency-update PRs

Renovate PRs are the highest-risk auto-merges in any system. Unit tests pass; runtime behavior breaks (transitive deps, lib regressions, framework patches, security-driven major bumps). Without QA on these PRs, breakages slip through unattended.

mqube doesn't address this — their Renovate flow is the default "build green = auto-merge" pattern. Leartech can do better by building on the existing self-hosted Renovate + Lighthouse trigger infra.

## What's already in place

From `~/leartech/hub/status/code-standards.md` (line 21):

> Renovate self-hosted on GCP only (scans all leartech repos Monday 7am UTC)

So Renovate already runs centrally; opens PRs across the leartech repo fleet weekly. Standard PR flow then runs whatever Lighthouse triggers fire (build, unit tests, scans).

What's NOT happening: full Playwright suite against a preview env. Renovate PRs go through the same shift-left-light path as any service PR with no test changes — whatever the trigger conditions allow.

## Two concrete moves

### Move 1: Renovate PRs run the full Playwright suite

Add a Lighthouse presubmit that's gated on the `dep-update` label (Renovate adds this by default). Triggers a full preview-env Playwright run.

```yaml
# .lighthouse/jenkins-x/triggers.yaml addition
presubmits:
- name: full-qa
  context: "full-qa"
  always_run: false
  optional: false
  source: "full-qa.yaml"
  queries:
  - labels:
      entries:
      - dep-update
    missingLabels:
      entries:
      - do-not-merge
      - do-not-merge/hold
```

```yaml
# .lighthouse/jenkins-x/full-qa.yaml — uses preview-shift-left harness
spec:
  pipelineSpec:
    tasks:
    - name: full-qa
      taskSpec:
        steps:
        - image: uses:spring-financial-group/leartech-pipeline-catalog/tasks/preview/spin-up.yaml@main
        - image: uses:spring-financial-group/leartech-pipeline-catalog/tasks/qa/playwright-full-suite.yaml@main
          # runs against the preview env; reads required-tests from leartech-qa-management
        - image: uses:spring-financial-group/leartech-pipeline-catalog/tasks/preview/tear-down.yaml@main
```

**Cost**: ~half day. Lighthouse trigger config, Tekton task composition. Reuses preview-shift-left harness.

### Move 2: Load-bearing dep blocklist requires human review

Some deps are too important to auto-merge ever — even with green tests. Auth libs, framework majors, DB drivers. Renovate's config supports this.

```yaml
# renovate.json5 in each repo (or shared config)
{
  packageRules: [
    {
      matchPackageNames: [
        "@auth0/auth0-spa-js",
        "github.com/spring-financial-group/leartech-go-common", // internal auth lib
        "react", "vue", "@angular/core",                         // framework majors
        "github.com/lib/pq", "go.mongodb.org/mongo-driver",      // DB drivers
        "express", "fastify", "gin",                              // web frameworks
      ],
      matchUpdateTypes: ["major"],
      automerge: false,
      addLabels: ["needs-human-review", "load-bearing-dep"],
    },
    {
      // default for other deps
      automerge: true,
      addLabels: ["dep-update"],
    },
  ],
}
```

The `load-bearing-dep` label triggers an additional `human-review-required` presubmit (always failing until manually overridden), forcing a CODEOWNERS sign-off.

**Cost**: ~2 hours per repo to wire the config. Bulk pass via a one-off PR-to-all-repos: ~half day.

## Combined flow

```
Monday 07:00 UTC → Renovate scans leartech repo fleet
                 → opens dep-update PRs across N service repos

For each PR:
  ├─ Standard presubmits fire: build, unit-tests, scans
  ├─ NEW: full-qa presubmit fires (gated on `dep-update` label)
  │   → spins up preview env
  │   → runs full Playwright suite from leartech-qa-management
  │   → results to result-store keyed by SHA
  │   → fails check if any required test fails
  ├─ If PR touches `load-bearing-dep` major version:
  │   → human-review-required check is failing
  │   → CODEOWNERS must `/override human-review-required` after manual review
  └─ If all green → auto-merge
```

## Failure-mode reasoning

What goes wrong in mqube's "no QA on Renovate PRs" world:

- A library upgrade subtly changes serialization → a test that should fail doesn't, because nobody runs the full suite → broken in prod
- A transitive dep changes timeout default → latency regression slips → prod p99 doubles silently
- A security patch fixes a CVE but changes a public method signature → caller breaks at runtime → null-pointer in prod

What this design catches:

- Full Playwright suite exercises real flows → serialization regressions surface
- (Phase 2) Load-test on Renovate PRs catches latency/perf regressions → SLA quill fails
- Human gate on load-bearing libs → CODEOWNERS evaluates breakage risk before merge

## Operational concerns

### Cost — running full QA on every Renovate PR

Preview env per Renovate PR + full Playwright suite is meaningfully more expensive than current cheap unit-test-only path. Mitigations:

1. **Batch Renovate PRs** — Renovate already supports grouped PRs (multiple deps together). Reduces total PR count.
2. **Skip full-QA on patch-version-only updates** — config rule:

   ```yaml
   - matchUpdateTypes: ["patch"]
     addLabels: ["dep-update", "skip-full-qa"]
   ```

   Only minor + major versions trigger the full suite. Patches go through standard checks.

3. **Tier preview-env aggressiveness** — full env only for `load-bearing-dep` repos; lighter env (single-service preview, mocked deps) for others.

### Time — dep-update PRs land slower

Full QA = ~10-30 min preview spin-up + Playwright. Acceptable for weekly cadence; not for every-15-min Dependabot patches. Renovate's batch-and-schedule means this lands once weekly anyway.

### Rollback — Renovate-introduced regressions

When a Renovate-merged change does break prod (which will happen), the rollback path is the same as any release: revert the helmfile entry. The audit log in `gate-metadata/overrides.log.jsonl` should capture if the regression slipped due to an override.

## Build estimate (with leverage)

| Piece | Estimate | Reuses |
|---|---|---|
| Lighthouse `full-qa` trigger + Tekton task | half day | leartech-pipeline-catalog `uses:` |
| `human-review-required` check + override config | 2 hours | Lighthouse `override` plugin |
| Renovate `packageRules` for load-bearing deps | half day | Renovate already self-hosted |
| Bulk apply Renovate config to existing repos | half day | one-off PR script |

Total: **~2 days**. Mostly config, not code.

## What this gets you that mqube doesn't have

- Renovate PRs subjected to the same QA bar as normal service PRs
- Load-bearing-dep major version bumps require human review (default-deny)
- Audit trail when overrides do happen — same `gate-metadata/overrides.log.jsonl`

## Open decisions

See `open-questions.md`:

- Which deps qualify as "load-bearing" — list needs to be curated, not just intuition. Worth a security-team review.
- Patch-version skip policy — too aggressive risks letting CVE patches with regressions through; too strict makes Renovate slow.
- Grouped PRs — bundle by category (auth deps, framework deps) or random?
