# `leartech-risk-assessor` — risk-based gating

A Tekton task that runs on every PR after preview spin-up. Classifies the PR's risk level by combining static analysis (primary signal) with Tempo + HAR coverage data (confirming signals), then writes a risk verdict to the result store. Consumed by the AI code reviewer (PR comment), the AI coverage scanner (companion PR proposals), and `leartech-gate` (override-approval requirements).

This is the answer to "uniform gating taxes every PR equally". A trivial isolated change should not pay the same coverage tax as a broad cross-cutting refactor. With risk-assessor, required test packs become **base + risk-modifier** rather than just **base**.

## Design principles

1. **Static analysis is the primary signal, not traces.** AST diff is environment-independent and deterministic — works regardless of preview-env coverage. Trace data is confirmation/refutation, not the foundation.
2. **The gate must function with zero trace data.** If Tempo is down, samples poorly, or the preview env is minimal, the assessor falls back to static-only with `confidence: low` and defaults to upper-bound risk. No silent failures.
3. **Rule-based first, ML later (maybe).** Explicit weights, explicit thresholds, explicit rationale. Developers see WHY their PR scored high, not just a number. ML only if the rule-based version proves insufficient.
4. **Risk levels map to test-pack modifiers**, not to gate pass/fail. The gate still gates on test results; risk modifies what tests are required.
5. **Audit trail is mandatory.** Every risk assessment writes its inputs, factor weights, and rationale to the result store. Developers can inspect the reasoning. Reviewers can challenge specific factors.

## Inputs

The risk-assessor consumes six signals, each with explicit confidence:

### 1. PR diff (static, free)

```
git diff main...HEAD --numstat
```

Files changed + line counts. Trivial.

### 2. AST analysis (static, primary)

Per-language tooling walks the changed functions and computes the **transitive impact set**:

| Language | Tool | What it produces |
|---|---|---|
| Go | `go-callvis` or custom `go/ast` walker | call graph: changed funcs → their callers → service ownership |
| TypeScript / Angular | `ts-morph` | same: changed exports → consumers → MFE/service ownership |
| Helm chart | `yq` + manual rules (CRDs/CronJobs/Services touched) | which K8s resources are affected |
| Rules config (YAML) | `yq` + ruleset cross-ref | which downstream rule consumers transitively affected |

Outputs a deduplicated list of services likely affected by the change. **Environment-independent** — runs in the Tekton pod with just the source checkout.

### 3. Service ownership map (static, declarative)

`leartech-qa-management/service-catalog.yaml` (new file in qa-management):

```yaml
services:
  - name: leartech-broker-ui
    repo: spring-financial-group/leartech-broker-ui
    type: ui-mfe
    owner: platform-broker
    consumes: [leartech-case-service, leartech-auth-service]
    consumed_by: []
    criticality: high              # high | medium | low — affects risk weights
  
  - name: leartech-case-service
    repo: spring-financial-group/leartech-case-service
    type: go-service
    owner: platform-case
    consumes: [leartech-orchestrator, leartech-rules]
    consumed_by: [leartech-broker-ui, leartech-platform-ui]
    criticality: high
```

Bootstrap source: AI-suggested from `.leartech/repo-type.yaml` + observed Tempo edges (one-time pass); CODEOWNERS confirm. Maintained going forward via release-time scanner (similar pattern to required-tests updates).

The catalog turns "function in repo X" into "service Y consumes services A, B, C", giving the AST tool what it needs to compute transitive impact.

### 4. Preview Tempo spans (confirming, optional)

During the Playwright run on the preview env, Tempo collects spans. After Playwright finishes, the assessor queries Tempo for spans in the run's time window:

```
GET /api/search?tags=preview_namespace=jx-leartech-broker-ui-pr-4521&start=...&end=...
```

Outputs: list of unique service-pair edges observed (`broker-ui → case-service`, `case-service → orchestrator`, etc.).

**This is the confirming signal**: if static says "should affect A, B, C" and Tempo shows "A, B, C, D were touched", D is an unexpected edge worth attention.

If Tempo is unavailable or sparse, this signal is dropped and the assessor records `trace_signal.confidence: low`.

### 5. Preview Playwright HAR (confirming, optional)

Same idea, different angle. Playwright HARs from the preview run reveal which **endpoints** were exercised. Useful for catching:

- New endpoints called that aren't in static analysis (could be dynamic dispatch, reflection, generated code)
- Endpoints that should have been hit but weren't (test gap exposed)

### 6. Baseline (last main SHA's risk-assessor output)

For change-detection: was this PR's set of affected services already covered by main, or has the scope expanded? Read from the result store at `gs://test-artifacts-product-first/results/v1/risk/<repo>/<main-sha>/...`.

Used for the **scope-delta factor**: are there new services in this PR that weren't already on main?

## Risk classification (rule-based v1)

Combine the inputs into a weighted score, then bucket into low/medium/high.

```python
# Pseudocode of the rule-based classifier
def classify(inputs):
    factors = []
    
    # Static factors (always available)
    services_affected = len(inputs.static.services_transitively_affected)
    factors.append(Factor("services_affected", services_affected, weight=lookup({1: "low", 2-4: "medium", 5+: "high"})))
    
    files_changed = inputs.diff.files_changed
    factors.append(Factor("files_changed", files_changed, weight=lookup({1-5: "low", 6-20: "medium", 21+: "high"})))
    
    if any(s.criticality == "high" for s in inputs.static.services_directly_affected):
        factors.append(Factor("touches_critical_service", True, weight="high"))
    
    # Trace factors (when available)
    if inputs.tempo.confidence != "low":
        unexpected_edges = set(inputs.tempo.edges) - set(inputs.static.predicted_edges)
        if unexpected_edges:
            factors.append(Factor("unexpected_trace_edges", len(unexpected_edges), weight="high"))
    
    # Scope-delta factor
    if inputs.baseline:
        new_services = set(inputs.static.services_transitively_affected) - set(inputs.baseline.services_transitively_affected)
        if new_services:
            factors.append(Factor("new_service_dependencies", len(new_services), weight="medium"))
    
    # Bucket
    high_count = sum(1 for f in factors if f.weight == "high")
    medium_count = sum(1 for f in factors if f.weight == "medium")
    
    if high_count >= 1:
        risk_level = "high"
    elif medium_count >= 2:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    # Confidence: low if trace signal was dropped, otherwise medium/high based on coverage
    confidence = "low" if inputs.tempo.confidence == "low" else "medium"
    
    return RiskAssessment(level=risk_level, confidence=confidence, factors=factors)
```

Tunable thresholds live in `leartech-qa-management/risk-config.yaml` — pull-driven like everything else.

## Output schema

Written to `gs://test-artifacts-product-first/results/v1/risk/<repo>/<sha>/<cluster>.json`:

```json
{
  "schema_version": "v1",
  "sha": "abc123def",
  "repo": "leartech-broker-ui",
  "cluster": "gcp",
  "assessed_at": "2026-04-30T14:23:01Z",
  "risk_level": "medium",
  "confidence": "medium",
  
  "static_signal": {
    "files_changed": 12,
    "lines_added": 320,
    "lines_removed": 87,
    "services_directly_affected": ["leartech-broker-ui"],
    "services_transitively_affected": [
      "leartech-broker-ui",
      "leartech-case-service",
      "leartech-orchestrator"
    ],
    "predicted_edges": [
      "leartech-broker-ui → leartech-case-service",
      "leartech-case-service → leartech-orchestrator"
    ]
  },
  
  "trace_signal": {
    "preview_env_coverage": "minimal",
    "preview_services_deployed": 3,
    "tempo_spans_observed": 1247,
    "tempo_edges_observed": [
      "leartech-broker-ui → leartech-case-service",
      "leartech-broker-ui → leartech-notification-service"
    ],
    "tempo_edges_unexpected_vs_static": [
      "leartech-broker-ui → leartech-notification-service"
    ],
    "tempo_edges_predicted_but_unobserved": [
      "leartech-case-service → leartech-orchestrator"
    ],
    "har_endpoints_new_vs_baseline": [
      "POST /api/cases/{id}/notify"
    ],
    "confidence": "medium"
  },
  
  "baseline_comparison": {
    "baseline_sha": "main-prev-sha-here",
    "new_service_dependencies": ["leartech-notification-service"],
    "removed_service_dependencies": []
  },
  
  "factors": [
    {"name": "services_affected", "value": 3, "weight": "medium"},
    {"name": "files_changed", "value": 12, "weight": "medium"},
    {"name": "unexpected_trace_edges", "value": 1, "weight": "high"},
    {"name": "new_service_dependencies", "value": 1, "weight": "medium"}
  ],
  
  "rationale": [
    "PR transitively affects 3 services per static analysis (medium scope)",
    "Tempo observed an edge to leartech-notification-service that static analysis did not predict — suggests an unexpected dependency was introduced; review for intent",
    "PR introduces a new service dependency vs main (notification-service)",
    "Preview env had minimal coverage (3 of 20 services deployed); trace signal confidence is medium not high"
  ],
  
  "recommended_test_packs": [
    "playwright-ui",
    "contract",
    "integration"
  ],
  
  "override_policy": {
    "approval_required": false,
    "two_key_required": false
  }
}
```

## Risk → coverage mapping

Extends `repo-type-policy/test-packs.yaml` from `qa-management-repo.md`:

```yaml
repo_types:
  go-service:
    required_packs: [unit, contract]
    risk_modifiers:
      low:
        # base required-packs only
      medium:
        add: [integration]
      high:
        add: [integration, dast]
        require_human_review: true
        require_two_key_override: true
  
  ui-mfe:
    required_packs: [playwright-ui, accessibility]
    risk_modifiers:
      low:
        # base only
      medium:
        add: [contract, bundle-size-budget]
      high:
        add: [contract, bundle-size-budget, lighthouse-perf]
        require_human_review: true
        require_two_key_override: true
```

The companion-PR pattern (from `qa-management-repo.md`) reads the risk assessment when proposing required-test additions: high-risk PR with new endpoint = propose stub + register + flag for human review.

## Consumers

### AI code reviewer

Cites the risk assessment in the PR comment:

```markdown
## leartech-risk-assessor: medium 🟡

**Static signal:** This PR transitively affects 3 services (broker-ui, case-service, orchestrator).

**Trace signal (medium confidence):** Tempo observed an edge to **notification-service** that static analysis did not predict. Review whether this dependency was intentional.

**Baseline comparison:** This PR introduces a new dependency on `leartech-notification-service` not present on main.

**Required test packs:** `playwright-ui`, `contract`, `integration` (medium-risk modifier added `integration`)

**Why medium:**
- 3 services affected (medium)
- 1 unexpected trace edge (high) ← contributing factor
- 1 new service dependency vs main (medium)

[Full assessment in result store →](https://...)
```

The reviewer doesn't tell the developer what to do; it gives them the audit trail to make an informed call.

### AI coverage scanner

When opening a companion PR to qa-management, includes the risk-modifier-derived test additions:

```yaml
# Companion PR proposes adding to required-tests/leartech-broker-ui.yaml:
required_tests:
  - name: notification-edge-flow                    # NEW
    blocking: true
    quarantined: false
    flake_budget: 0.05
    owner: platform-broker
    suite: integration
    rationale: "Required by risk-assessor v1.2.0 — PR introduces new edge to notification-service"
```

### `leartech-gate` (porcupine-equivalent)

Reads the latest risk assessment for the SHA being promoted. Applies override-policy from the assessment:

- `override_policy.approval_required: true` → `/override leartech-gate` requires approval comment from a different actor than the override caller
- `override_policy.two_key_required: true` → requires two distinct approvers

Both enforced via Lighthouse webhook plugin or audit-log review.

## How this addresses the preview-coverage problem explicitly

The user-raised concern: "preview just doesn't have as much service coverage; how do we know if the PR is broader or just under-tested?"

Three architectural responses, layered:

1. **Static signal is primary.** AST-derived scope doesn't depend on preview coverage. If static says "this affects 3 services", that's load-bearing regardless of what the preview env shows.

2. **Trace signal records its confidence.** When preview is minimal, `trace_signal.preview_env_coverage: minimal` and `confidence: medium` (not high). The classifier weights the trace factors lower in this case.

3. **Default to upper-bound when uncertain.** If both static and trace signals are inconclusive (e.g. dynamic dispatch hides scope, preview is bare), the classifier defaults to `risk_level: medium` rather than `low`. Conservative.

This means a PR can never be "false-low" because the env was empty. Worst case, it's medium when it could've been low — which is acceptable; over-testing is a smaller harm than under-testing.

## Build estimate (with leverage)

| Piece | Effort | Reuses |
|---|---|---|
| AST analysis Tekton task — Go (`go-callvis` wrapper) | 4 days | golden Go template; preview-shift-left for iteration |
| AST analysis Tekton task — TypeScript (`ts-morph`) | 3 days | leartech-pipeline-catalog `uses:` pattern |
| Service ownership map (`service-catalog.yaml` + bootstrap script) | 3 days | AI classifier; `repo-type.yaml` already in repos by Phase 1 |
| Tempo span collector (extends `tempo-to-har`) | 2 days | Phase 2's tempo-to-har already built |
| HAR endpoint extractor | 1 day | Phase 2's HAR pipeline already built |
| Risk classifier (Go service, rule-based) | 4 days | `leartech-go-common`; data-driven thresholds |
| Result-store schema + write | 2 hours | existing `gs://test-artifacts-product-first/` extended |
| AI code-reviewer integration (cite risk in PR comment) | 1 day | extends `leartech-ai-classifier` prompt |
| Companion-PR risk-derived test additions | 1 day | extends Phase 2's coverage scanner |
| Gate override-policy enforcement | 1 day | extends Phase 1 gate |
| Documentation + runbook | 1 day | — |

Total: **~17 person-days = ~3 weeks** for one engineer; ~2 weeks parallelized across 2.

Position in build plan: **Phase 2.5** — after Phase 2 (which provides the tempo-to-har + HAR pipeline + AI coverage scanner) and before any optional Phase 3 items. Worth a separate phase because it's substantial and deserves dedicated attention; not just a Phase 2 line-item.

## What I'd warn against

### Don't start with ML

Tempting because "risk score" sounds quantitative and ML feels native. But:

- Training data lifecycle: need labeled examples ("this PR was risky and broke prod" vs not). Labels take weeks to land. Cold-start problem severe.
- Explainability: "your PR scored 0.73" is unhelpful. "Your PR touches 3 critical services and introduces a dependency main doesn't have" is actionable.
- Drift: codebase shape changes over time; ML model drifts; constant retraining; ops burden.

Rule-based classifier with explicit factors and tunable weights is interpretable, auditable, and easy to evolve. Move to ML only if:
- The rule-based version proves insufficient AFTER tuning
- You have >6 months of result-store data
- You have labeled outcomes (PRs tagged as "caused regression" or not)

### Don't make trace signal mandatory

If the gate requires trace data and Tempo is sampling poorly, the gate becomes flaky → developers learn to retry rather than understand → trust erodes. Static signal must stand alone; trace is supplementary.

### Don't auto-merge based on risk

Even a `low` risk doesn't justify auto-merge of arbitrary PRs. Risk scoring informs **what's required**, not **whether to merge**. The gate decision still requires the required tests to pass; risk just changes which tests are required.

### Don't expose the score raw

`risk_level: medium` is fine. `risk_score: 0.847` is not — it implies precision the model doesn't have. Bucketed levels with rationale > continuous scores. Easier for humans to act on.

## Calibration feedback loop with arrivals-observer (Phase 2.7)

A subtle but valuable closing of the architectural loop: when `leartech-arrivals-observer` (Phase 2.7) detects a regression, its traffic-forensics output reveals which **service-edges actually changed** between the regressing version and the prior good version. Comparing those revealed edges against risk-assessor's static prediction set surfaces blind spots in the static analysis.

```
Risk-assessor at PR-time predicted: PR affects A, B, C
Arrivals-observer at deploy-time detects: regression in B's tests
Traffic-forensics on the regression: edge A → notification-service appeared,
                                     notification-service was NOT in static prediction
                                     → risk-assessor missed an edge
```

Track as `unpredicted_edge_rate` metric in Phase 2.7 retros:

```
unpredicted_edge_rate = forensics-revealed-edges-not-in-static-prediction / total-regressions
```

If consistently >20%, risk-assessor's static rules need improvement. Common causes:

- **Shared config repo effects under-modeled** — a config change reaches more services than the AST shows (rule-engine consumers, message-bus subscribers)
- **Dynamic dispatch / reflection / DI** — Go interfaces, dependency injection, generated client code; static walk doesn't follow these
- **Generated code consumers** — AST sees generation source but not the generated callers (OpenAPI clients, protobuf, etc.)
- **Message-bus / event-driven decoupling** — event publisher's static deps don't reach subscribers

Each pattern, once identified, becomes a new rule in `service-catalog.yaml` (e.g. mark a service as a transitive consumer of a shared config) or `risk-config.yaml` (e.g. raise risk weight when a known-tricky shared dep changes). **The system gets smarter via operational feedback** rather than via hand-tuning every quarter.

This closes a real gap: static analysis fundamentally can't see what runtime behavior reveals; the loop lets it learn.

## Open decisions

See `open-questions.md`. Risk-assessor-specific:

- Q21: Service ownership map bootstrap — AI-suggested + CODEOWNERS-confirm vs. manual seed?
- Q22: Confidence-vs-risk trade-off — when confidence is low, default to medium or high?
- Q23: Cross-cluster risk — does the PR's risk vary by cluster (gcp vs az), or is it cluster-agnostic?
- Q24: Risk-config governance — how often are thresholds tuned? Who owns?
