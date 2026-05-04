# Sessions plan + parallelization + live status

The execution-tracking artifact for building this architecture. Three views:

1. **Linear session plan** — discrete, end-to-end-testable units
2. **Parallelization plan** — which sessions can fan out across multiple Claude Code sessions
3. **Live status** — running log, updated as sessions complete

Following the leartech "session 11/12" pattern from `~/leartech/hub/status/code-standards.md`.

---

## Part 1: Linear session plan

Each session aims for **3-6 hours of focused work** producing a **testable, committable deliverable**. Skip a session if its deliverable can't be defined cleanly — that's a sign the design needs another pass; update `open-questions.md` instead.

### Phase 1 — closed loop (~7 sessions)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **1.1** | Stand up `leartech-qa-management` repo | Public GH repo with CUE schemas, self-CI (schema-validate + cross-reference linter), seed entries for 3-4 sample services, Renovate-publishable | new repo: `mikelear/leartech-qa-management` | 4-5h |
| **1.2** | Extend result-store path schema | `end2end-ui` task uploads `results.json` to SHA-keyed `gs://test-artifacts-product-first/results/v1/...`; verified on a real PR | `leartech-pipeline-catalog` | 2-3h |
| **1.3** | `leartech-gate` skeleton + first quill | Go service from golden template; helmfile parse via beaver lib; GCS result-store client; **shift-left-tests quill** end-to-end against synthetic helmfile + results | new repo: `mikelear/leartech-gate` | 4-6h |
| **1.4** | Two more quills + Tekton wrapper | `contract-tests` + `co-promotion` quills; pipeline-catalog `uses:` task; GitHub PR comment + check status post | `leartech-gate` + `leartech-pipeline-catalog` | 4-5h |
| **1.5** | First-real-world wiring | Lighthouse trigger + branch-protection on a target GitOps repo (probably the dev cluster's first); green check on a real PR; synthetic failure properly blocks | a GitOps repo (TBD which first — `jx-staging-files` likely) | 3-4h |
| **1.6** | `repo-type.yaml` bulk seed | AI-suggested classification PR per active repo; CODEOWNERS confirm; 100% coverage of active repos | many repos (one PR each) | 4-5h spread across ~3 sessions if needed |
| **1.7** | Phase 1 retrospective | Validation criteria checked; `~/leartech/hub/shared-rules/platform-leverage.md` started with patterns landed; Phase 2 estimates recalibrated from actuals | hub | 1-2h |

**Phase 1 total**: ~22-30 hours of focused work; calendar-wise ~2-3 weeks single-threaded, ~1.5 weeks with parallelization.

### Phase 2 — enrichment (~7 sessions)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.1** | HAR pipeline foundations | Playwright records HAR explicitly; upload to `har/v1/...`; sanitize utility forked from mqube + adapted | `leartech-pipeline-catalog` + new: `leartech-go-common/sanitize` | 3-4h |
| **2.2** | `leartech-load-testing` service | Go service from template; replay engine; OAuth substitution; Mongo + GCS write; API endpoints; chart | new repo: `mikelear/leartech-load-testing` | 6-8h |
| **2.3** | SLA quill + load configs | `load-sla` quill in gate (alert-only); `load/<service>.yaml` schema in qa-management; nightly CronJob generator | `leartech-gate` + `leartech-qa-management` + GitOps repos | 4-5h |
| **2.4** | `tempo-to-har` synthesizer | Go service that queries Tempo and emits HAR; chart; nightly schedule | new repo: `mikelear/tempo-to-har` (or as part of `leartech-load-testing`) | 3-4h |
| **2.5** | AI coverage scanner + companion PRs | Tekton task; cross-repo PR dependency wiring; auto-merge for additions; human review for removals | `leartech-pipeline-catalog` + integrations with `leartech-ai-classifier` | 5-7h |
| **2.6** | Renovate hardening | `full-qa` Lighthouse trigger; Renovate `packageRules` for load-bearing deps; bulk apply across repos | many repos + central Renovate config | 3-4h |
| **2.7** | Phase 2 retrospective | Override-rate analysis from result store; flake-budget tuning pass; platform-leverage register updated | hub + `leartech-qa-management` | 2h |

**Phase 2 total**: ~26-34 hours; calendar ~2 weeks parallel.

### Phase 2.5 — risk-based gating (~7 sessions)

| # | Goal | Deliverable | Where commits land | Est |
|---|---|---|---|---|
| **2.5.1** | `service-catalog.yaml` bootstrap | AI-suggested catalog entries for every active service; CODEOWNERS-confirmed PRs | `leartech-qa-management` | 4-5h |
| **2.5.2** | AST analyzer — Go | `go-callvis` wrapper or custom `go/ast` walker; Tekton task; outputs predicted-impact JSON | `leartech-pipeline-catalog` (or new `leartech-ast-analyzer` if it grows) | 6-8h |
| **2.5.3** | AST analyzer — TypeScript | `ts-morph` wrapper; same output schema | same | 5-6h |
| **2.5.4** | `risk-assessor` core | Tempo span collector; HAR endpoint extractor; rule-based classifier; result-store write of risk verdict | new repo or `leartech-pipeline-catalog` task | 6-8h |
| **2.5.5** | Gate integration + override stiffening | `risk-override` quill in `leartech-gate`; approval-required logic via Lighthouse webhook; audit log writer | `leartech-gate` | 3-4h |
| **2.5.6** | AI integration — risk in PR comments | Code-reviewer cites risk + factors; coverage-scanner uses risk-modifiers when proposing companion PR test additions | `leartech-pipeline-catalog` + `leartech-ai-classifier` integration | 4-5h |
| **2.5.7** | Phase 2.5 retro + threshold tuning | False-positive rate measurement on first 30 PRs; threshold tuning in `risk-config.yaml`; documentation | `leartech-qa-management` + hub | 2-3h |

**Phase 2.5 total**: ~30-39 hours; calendar ~3 weeks (some sessions hard to parallelize).

### Phase 3 — strategic / conditional

Each item is independent and gated on operational signal. **Don't pre-plan**; create a session when the trigger fires.

| Trigger condition | Session | Est |
|---|---|---|
| Tempo coverage gaps measurable in 2.5.7 retro | Pixie deploy + `pixie-to-har` exporter | ~1 week |
| First production-only regression slips through | Synthetic prod probes (Tekton CronJob) | ~1 week |
| Team explicitly wants Slack signal | Slack-alert layer (small Go service reading result-store) | ~3 days |
| Coverage-gap report shows persistent untested edges | Test-generation AI | ~3 days |
| Pact contract testing adopted | Contract-derivation tool | ~1 week |
| Security team wants automated fuzzing | Security-replay extension to `leartech-load-testing` | ~1-2 weeks |
| Cronjob-heavy services emerge | Cronjob health quill | ~3 days |

---

## Part 2: Parallelization plan

The bottleneck isn't Claude session count; it's **the contracts between sessions**. Once a contract is committed (a schema, an API, a file path), parallel sessions can consume it without coordinating.

**The rule**: commit the contract first, then fan out.

### Phase 1 dependency graph

```
1.1 (qa-management schemas) ─┬─► 1.3 (gate uses schemas) ─► 1.4 ─► 1.5 ─► 1.7
                             ├─► 1.2 (result-store extends path)
                             └─► 1.6 (repo-type.yaml seed)
```

Once 1.1 commits the CUE schemas + path conventions (probably half a day in), three streams can fan out:

| Stream | Sessions | Why parallel |
|---|---|---|
| A | 1.1 → 1.3 → 1.4 → 1.5 → 1.7 | The gate is the critical path |
| B | 1.2 (result-store extension in pipeline-catalog) | Different repo, settled contract |
| C | 1.6 (bulk repo-type.yaml seed) | Different repos, settled schema |

**Parallel saving**: ~3-4 days off Phase 1 with two engineers; ~5 days with three.

### Phase 2 dependency graph

```
After Phase 1:
   ├─► 2.1 → 2.2 → 2.3 (HAR + load testing + SLA quill)
   ├─► 2.4 (tempo-to-har, independent)
   ├─► 2.5 (AI coverage scanner, independent)
   └─► 2.6 (Renovate hardening, mostly config)
```

**Four parallel streams** is realistic. The integration retro (2.7) merges learnings; until then each stream owns its own repo + Tekton task.

**Parallel saving**: ~1 week off Phase 2 with two engineers.

### Phase 2.5 dependency graph

```
   ├─► 2.5.1 (service-catalog bootstrap, independent)
   ├─► 2.5.2 (Go AST analyzer)
   ├─► 2.5.3 (TS AST analyzer — same pattern, different language)
   └─► 2.5.4 (risk-assessor core — stubs AST inputs initially)
   
   then converge:
   ├─► 2.5.5 (gate integration)
   └─► 2.5.6 (AI reviewer integration)
```

**Parallel saving**: ~1.5 weeks off Phase 2.5 with two engineers.

### Mechanics — running multiple Claude Code sessions

#### Terminal layout

- **One pane per session** in tmux / iTerm splits
- **One repo per session** — different working directories
- **Name sessions clearly** — title bar set to "qa-arch / 1.2 result-store" etc.

#### Working directories

Each session needs its own checkout. Two patterns:

1. **Different repos** — natural separation. `~/leartech/leartech-gate/` and `~/leartech/leartech-load-testing/` don't conflict.
2. **Same repo, different branches** — use `git worktree`:
   ```bash
   cd ~/leartech/leartech-gate
   git worktree add ../leartech-gate-quill-2 feature/contract-quill
   ```
   Each worktree is an independent checkout sharing the same `.git`. Two sessions can work on different branches without stepping on each other.

#### Coordination via hub

The hub becomes the bulletin board. Each session, on completing meaningful work, updates `~/leartech/hub/status/qa-architecture.md` with a one-liner. The next session that starts can `cat` the file to know ground state.

See the live-status section below for the format.

#### Per-session brief

Each Claude Code session starts with a focused brief naming:

- The session ID and goal
- Pre-requisites (other sessions that must be done first; commits / tags they should pull)
- Exact deliverable
- Files to read first
- Things NOT to do (preventing scope creep)

Example template:

```
Session 1.3: leartech-gate skeleton + first quill

Pre-reqs: Phase 1 session 1.1 done (schemas committed at qa-management
v0.1.0); leartech-go-service-template available.

Deliverable: 
- Go service in mikelear/leartech-gate
- Helmfile parse via beaver lib
- shift-left-tests quill green against synthetic helmfile + GCS results
- Tekton task wrapper in pipeline-catalog with `uses:` pattern

Read first:
- ~/leartech/qa-architecture/gate.md
- ~/leartech/qa-architecture/qa-management-repo.md (schema)
- mikelear/leartech-qa-management at v0.1.0

Don't:
- Add multiple quills (that's session 1.4)
- Wire to a real GitOps repo (that's session 1.5)
```

### Disciplines that make parallelization work

Five things matter more than session count:

1. **Contract-first commits.** Before fanning out, commit the load-bearing contract. For Phase 1, that's the qa-management schemas + result-store path. For Phase 2, the HAR storage path schema. For Phase 2.5, the risk-verdict JSON schema. **No contract → no parallelism.**

2. **Small frequent commits.** Each session commits every 30-60 min. Other sessions pull frequently. Late merges are the killer.

3. **One repo, one session.** Don't run two sessions in the same working dir. Use worktrees if you must, separate repos otherwise.

4. **Hub status as bulletin board.** Update on completion of each session; readable in 30s by another session.

5. **The human as router.** With 3 sessions running, the human triages: reads outputs, spots cross-session conflicts, redirects when sessions wander. **Don't run more than 3 simultaneously** — the triage burden grows quadratically.

### What NOT to parallelize

- **Initial repo creation** — `gh repo create` is one-and-done; do it serially.
- **Schema-defining sessions** — single source of truth means single author at a time.
- **Cross-cutting refactors** — when extracting to `leartech-go-common`, one session at a time on the shared lib.
- **The retrospective** — bring streams together, don't fan out the analysis.

### Realistic calendar with parallelism

```
Week 1
  Stream A: 1.1 (schemas, day 1) → 1.3 → 1.4
  Stream B: (waits for 1.1) → 1.2
  Stream C: (waits for 1.1) → 1.6 (started; 30 PRs takes ~3 days elapsed)

Week 2
  Stream A: 1.5 → 1.7 (retro, brings everyone in)
  Stream C: continues 1.6 to completion

Week 3
  Stream A: 2.1 → 2.2 → 2.3
  Stream B: 2.4
  Stream C: 2.5
  Stream D: 2.6 (small, finishes mid-week)

Week 4
  All streams converge → 2.7 retro

Week 5
  Stream A: 2.5.1 (catalog bootstrap)
  Stream B: 2.5.2 (Go AST)
  Stream C: 2.5.3 (TS AST)
  Stream D: 2.5.4 (risk-assessor core, stubs AST)

Week 6
  Stream A: 2.5.5 (gate integration)
  Stream B: 2.5.6 (AI reviewer integration)
  Convergence → 2.5.7 retro
```

That's **~6 weeks of calendar time vs ~7 single-threaded** with two engineers; **~5 weeks** with three at peak Phase 2 / 2.5 fan-out.

The bottleneck switches from "engineering capacity" to "human triage capacity" pretty quickly. Two parallel sessions is sustainable; four is intense; six is unsustainable.

---

## Part 3: Live status

Updated as sessions complete. Format: one row per session, with a one-line note on what landed. Sessions in flight tagged `🚧`; completed `✅`; blocked `⛔`.

### Phase 1

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 1.1 | ⏳ | not started | — |
| 1.2 | ⏳ | not started | — |
| 1.3 | ⏳ | not started | — |
| 1.4 | ⏳ | not started | — |
| 1.5 | ⏳ | not started | — |
| 1.6 | ⏳ | not started | — |
| 1.7 | ⏳ | not started | — |

### Phase 2

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 2.1 | ⏳ | not started | — |
| 2.2 | ⏳ | not started | — |
| 2.3 | ⏳ | not started | — |
| 2.4 | ⏳ | not started | — |
| 2.5 | ⏳ | not started | — |
| 2.6 | ⏳ | not started | — |
| 2.7 | ⏳ | not started | — |

### Phase 2.5

| # | Status | Note | Commit / PR |
|---|---|---|---|
| 2.5.1 | ⏳ | not started | — |
| 2.5.2 | ⏳ | not started | — |
| 2.5.3 | ⏳ | not started | — |
| 2.5.4 | ⏳ | not started | — |
| 2.5.5 | ⏳ | not started | — |
| 2.5.6 | ⏳ | not started | — |
| 2.5.7 | ⏳ | not started | — |

### Active sessions (live multi-session view)

When parallel sessions are running, list them here so each session can see ground state at a glance.

```
(none active)
```

Format when active:

```
- 1.3 gate-skeleton — in progress on `mikelear/leartech-gate@feat/skeleton`; first quill green
- 1.2 result-store — done; `pipeline-catalog` v0.5.0 cut; consumers can pin
- 1.6 repo-type-seed — 12/30 repos PR'd; CODEOWNERS approving
```

### Lessons / leverage register

Patterns that landed during sessions, with the marginal cost of next instance. Move stable entries to `~/leartech/hub/shared-rules/platform-leverage.md` once Phase 1 ships.

| Pattern landed | Reusable for | Marginal cost |
|---|---|---|
| _(none yet)_ | | |

---

## How to use this document

**As a planner:**
- Sessions table tells you what's next
- Parallelization view tells you which streams can run concurrently
- Calendar tells you realistic elapsed time

**As a session:**
- Check live-status before starting — know what's in flight
- Update live-status after committing — let other sessions see ground state
- Refer to relevant doc(s) named in the "Read first" section of the brief
- Keep within scope; defer surplus to a later session

**As the human router:**
- 1-2 parallel sessions is sustainable
- 3 is intense but workable
- 4+ becomes triage burden — slow down
- Daily 15-min review of live-status keeps streams aligned
