# QA Architecture — Leartech

This workspace is a **forward design** for leartech's QA + release-gating system. It builds on:

- **mqube reverse-engineering** at `~/leartech/Qa-Analysis/` — the reference pattern (Fat Controller / Porcupine / automated-qa-service / architect / load-testing). Read those findings first if you don't already know how mqube works.
- **Leartech's existing foundation** — golden templates, shared pipeline catalog, preview-shift-left harness, AI code-review + classifier, self-hosted Renovate, multi-cluster JX3, Grafana stack (Tempo / Loki / Prometheus), all already running on both clusters.

The goal is a leartech QA system that takes the best of mqube but improves on its weaknesses: single source of truth for test config (no three-place drift), PR-time gating (not post-deploy), pull-based config propagation (Renovate-driven), HAR as a multi-consumer interchange format, and an OTel-first traffic-mapping path that reuses Tempo before adding new infra.

## When a session is running here

Your job is to either (a) advance the design (write or refine these docs), (b) advise on implementation as pieces start landing, or (c) update for new decisions. The build plan in `build-plan.md` is the source of truth for what's next.

## Core design principles

1. **Single source of truth for QA config** — `leartech-qa-management` repo. All other consumers (gate, test runners, load tester) pull from it via Renovate-pinned tags. No fan-out / push.
2. **PR-time gating, not release-time** — coverage gaps caught before merge, when the author has full context. Release-time and weekly cron are cleanup, not gates.
3. **Go for new services** — mirrors K8s ecosystem, golden template exists (`leartech-go-service-template`), reuses `leartech-go-common`.
4. **HAR as interchange format** — multiple producers (Playwright, Tempo, optionally Hubble/Pixie later), single sanitize/template layer, single replay engine.
5. **Tempo first for traffic mapping** — leartech already has it; only add Pixie/Hubble if Tempo coverage is genuinely insufficient.
6. **Repo-type-driven test policy** — `.leartech/repo-type.yaml` per repo + a hub shared rule maps types to required test packs. Code reviewer enforces.
7. **Foundation leverage is the budget** — estimates assume golden templates / pipeline-catalog / AI classifier / shared-rules registry are reused. Track marginal cost of each new pattern in `~/leartech/hub/shared-rules/platform-leverage.md` (proposed — not yet created).

## Files

@README.md
@qa-management-repo.md
@gate.md
@har-pipeline.md
@risk-assessor.md
@arrivals-observer.md
@notifications.md
@renovate-hardening.md
@build-plan.md
@sessions.md
@session-0-brief.md
@session-0-lessons.md
@session-0c-brief.md
@session-2-4-brief.md
@open-questions.md
@diagrams.md

## Cross-references

- `~/leartech/Qa-Analysis/findings/06-porcupine.md` — the gate pattern we're adapting
- `~/leartech/Qa-Analysis/findings/01-automated-qa-service.md` — the test runner we're adapting
- `~/leartech/Qa-Analysis/findings/03-release-gate.md` — what mqube actually does (corrected, third version)
- `~/leartech/hub/shared-rules/golden-service-standard.md` — leartech observability stack already in place
- `~/leartech/automated-agent/` — separate workspace; auto-PR system. Phase 2.7's arrivals-observer integrates via `LessonCaptureNotifier` (see `notifications.md`).

## Cross-system integration: `~/leartech/automated-agent/`

When QA-analysis findings should propagate to the automated-agent's calibration system:

- **Manual curation** (deep-dive findings): use `~/leartech/automated-agent/gate/agent/lessons/cli.py` directly with `--source-type {staging_test|prod_incident|manual_review}` and `--status open`. For sessions looking deeply at a specific issue.
- **Auto-capture** (Phase 2.7+): `arrivals-observer` fires `LessonCaptureNotifier` (a transport in the `notify` framework — see `notifications.md`) when the suspected PR author is the configured `automated-agent-bot` identity. Auto-captured lessons land as `--status candidate` for human triage; not in the active calibration queue until promoted to `open`.

Don't modify `~/leartech/automated-agent/` from this workspace — it's a separate concern. Cross-references only.

## Conventions

- Cite source files with line numbers when describing existing systems (`file.go:42`).
- Distinguish **proposed** (marked as such) from **decided** (no caveat).
- When a decision is made, move it from `open-questions.md` into the relevant doc and remove from the questions list.
- Keep schemas concrete (paste full YAML examples) — abstract descriptions decay; concrete examples are testable.
