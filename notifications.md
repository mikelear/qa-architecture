# Notifier framework — pluggable transports for QA events

A small abstraction in `leartech-go-common/notify/` that decouples "something interesting happened" from "send it to Slack". One interface, many transports, configuration-driven routing in `leartech-qa-management`. Reusable across every QA-system component that needs to alert humans or downstream systems.

This is the structural fix that lets us swap transports (Teams, email, PagerDuty), disable notifications entirely (NoopNotifier), or fan a single event out to multiple destinations — without rewriting consumers.

## Why this exists

Mqube's Fat Controller hard-codes a Slack alerter. We've inherited that pattern in early Phase 2.7 drafts. Two problems with that:

1. **Lock-in to Slack** — if leartech moves off Slack to Teams, or wants to add PagerDuty for high-severity, it's a rewrite each time.
2. **Multiple consumers, multiple Slack integrations** — when arrivals-observer + leartech-gate + risk-assessor each want to alert, we'd duplicate webhook handling, userMappings lookup, and rendering. The notification surface area grows; the abstraction pays off the more it's used.

The Notifier interface is also the cleanest place to land **the automated-agent lesson capture integration** (per `~/leartech/automated-agent/`) — auto-capture becomes another transport, filtered by PR-author, with its own routing rules. No bespoke wiring per consumer.

## Where it lives

`~/leartech/leartech-go-common/notify/` — new package, lives alongside the existing `pkg/auth`, `pkg/logger`, `pkg/lock`, etc. Reusable across all leartech Go services.

## Core interface

```go
// notify/types.go

package notify

import (
    "context"
    "time"
)

type Severity string

const (
    SeverityInfo  Severity = "info"
    SeverityWarn  Severity = "warn"
    SeverityError Severity = "error"
)

// Identity represents a user across multiple platforms.
// Resolved from leartech-qa-management's notification-config.yaml.
type Identity struct {
    GitHub string            // canonical handle
    Other  map[string]string // {slack: U..., teams: 28:..., email: x@..., pagerduty: PXXX}
}

// Event is the common payload across all notifications.
// Producers (arrivals-observer, gate, risk-assessor, ...) build these;
// transports (SlackNotifier, etc.) render and dispatch.
type Event struct {
    Type     string             // dotted: "arrivals.regression", "arrivals.timeout", "gate.override", "risk.high"
    Severity Severity
    Time     time.Time
    Service  string             // affected service (if applicable)
    Version  string              // affected version (if applicable)
    SHA      string              // affected commit (if applicable)
    Cluster  string              // gcp / az (if applicable)
    Author   *Identity           // suspected PR author (if applicable)
    Title    string              // one-line summary
    Body     string              // structured details — markdown
    Links    map[string]string   // {pr: url, trace: url, results: url, ...}
    Tags     map[string]string   // free-form metadata for filtering / routing
}

type Notifier interface {
    Notify(ctx context.Context, event Event) error
}
```

## Built-in implementations

### `SlackNotifier` (Phase 2.7 v1)

```go
// notify/slack.go
type SlackNotifier struct {
    WebhookURL  string                       // from a K8s secret
    UserLookup  func(github string) *Identity // resolves GitHub→Slack ID
    Channel     string                        // default channel ID
    FallbackTag string                        // "@here" if user lookup fails
    Renderer    SlackRenderer                 // optional custom message rendering
}
```

Posts to a Slack webhook. Renders the Event as a Slack-formatted message with PR-author @-mention (using `Author.Other["slack"]`), failed tests list, traffic forensics diff (when present in `Tags`), and Links section. Falls back to `@here` when user lookup misses — no silent skip (this fixes the mqube-fat-controller pattern that silently skipped non-mapped authors).

### `LessonCaptureNotifier` (Phase 2.7 v1.1, Phase 3 if hot)

```go
// notify/lesson_capture.go
type LessonCaptureNotifier struct {
    AgentRoot       string                  // ~/leartech/automated-agent (path or HTTP API URL)
    SourceTypeMap   map[string]string       // event.Tags["env"] → "staging_test" / "prod_incident"
    DefaultStatus   string                  // "candidate" — NOT "open"; humans triage
    DefaultObserver string                  // "arrivals-observer"
}
```

Calls automated-agent's lesson-capture API (CLI initially, HTTP endpoint later) when fired. Lesson lands as `status: candidate` so it doesn't enter the active calibration queue until a human (or designated bot) reviews and promotes to `open`. Manual capture path stays available for QA-analysis deep-dives — see `~/leartech/automated-agent/gate/agent/lessons/cli.py`.

**Critical**: this transport is **only** wired in via routing rules that filter on `Author.GitHub == automated-agent-bot` (or whatever the agent's commit identity is). A regression caused by a human PR does NOT generate an agent-lesson candidate. See routing config below.

### `WebhookNotifier` (generic)

```go
// notify/webhook.go
type WebhookNotifier struct {
    URL        string                   // from a K8s secret
    Headers    map[string]string         // bearer token, etc.
    PayloadFn  func(Event) ([]byte, error)  // custom rendering (default: JSON)
}
```

POSTs Event JSON (or custom rendering) to a configured URL. Catch-all for future integrations (PagerDuty, Linear, Jira, custom dashboards) without writing a new impl per service.

### `NoopNotifier`

```go
// notify/noop.go
type NoopNotifier struct {
    Logger logger.Logger // optional — logs the event but doesn't dispatch
}
```

Used in dev / testing / when an event type is configured with no transports. **Critically also the answer to "what if leartech doesn't have Slack at all"** — set all transports to `noop` and notifications still fire (visible in pod logs + result-store metadata) without depending on any external service.

### `FanoutNotifier`

```go
// notify/fanout.go
type FanoutNotifier struct {
    Notifiers []Notifier
}
```

Multiplexes a single Event to multiple transports. Used when routing config specifies multiple destinations (e.g. `[slack, lesson-capture]`). Errors from individual transports are logged but don't abort the others — best-effort dispatch.

### Future: `TeamsNotifier`, `EmailNotifier`, `PagerDutyNotifier`

Add when needed. Same interface; ~half day each. The framework's value is that adding a new transport is local — no consumer changes.

## Configuration in `leartech-qa-management`

Single source of truth, pull-based, Renovate-pinned (matches the other qa-management config patterns).

```yaml
# leartech-qa-management/notification-config.yaml

transports:
  slack:
    webhook_secret_ref: slack-webhook       # K8s secret name in consumer's namespace
    default_channel: "C0LEARTECH_RELEASES"
  
  lesson-capture:
    agent_root: /var/run/leartech-automated-agent  # in-cluster mount or HTTP URL
    default_status: candidate
    default_observer: arrivals-observer
  
  pagerduty-prod:
    webhook_secret_ref: pagerduty-prod-webhook
  
  teams:                                    # placeholder; populate when adopted
    webhook_secret_ref: teams-webhook
  
  noop: {}                                  # always available

users:
  mikelear:
    github: mikelear
    slack: U02ABCDEF
    teams: 28:abc-..-def
    email: mike.lear@leartech.com
  craggs:
    github: craggs
    slack: UHE67AHS7
    email: craggs@leartech.com
  # ... per-user dict; each user has identifiers per platform they use

automated_agents:
  github_handles:
    - automated-agent-bot                   # whatever the agent's commit identity is
    - leartech-bot                          # any other bot identities to track

routing:
  arrivals.regression:
    transports: [slack]                                 # always — humans see all regressions
    fallback_tag: "@here"
    
    conditional_transports:
      - when:
          author.is_automated_agent: true               # filter: PR author matches automated_agents list
        transports: [lesson-capture]                     # ALSO fire lesson-capture for agent-PRs
        config:
          source_type: ${event.tags.env_classification}  # "staging_test" or "prod_incident"
  
  arrivals.timeout:
    transports: [slack]
    fallback_tag: "@here"
  
  gate.override:
    transports: [slack]
    only_when:
      severity: error                       # only fire on high-risk overrides
  
  risk.high:
    transports: []                          # currently silent; flip to [slack] if signal becomes useful
  
  ai-review.score-low:
    transports: [slack]
    only_when:
      score: "<60"
  
  renovate.full-qa-fail:
    transports: [slack]
```

## How a consumer uses this

```go
// arrivals-observer/cmd/api/main.go (sketch)

import (
    "github.com/spring-financial-group/leartech-go-common/notify"
)

func main() {
    // Load notification-config.yaml from leartech-qa-management
    cfg := loadNotificationConfig(...)
    
    // Build a router that selects transports per event-type
    router := notify.NewRouter(cfg)
    
    // Inject into the arrivals usecase
    arrivalsUseCase := arrivals.NewUseCase(
        arrivalsRepo,
        router,                          // <-- replaces specific Slack alerter
        gitClient,
        ...
    )
}

// Inside arrivals/usecase.go on regression detection:
event := notify.Event{
    Type:     "arrivals.regression",
    Severity: notify.SeverityError,
    Service:  arrival.ServiceName,
    Version:  arrival.Version,
    SHA:      arrival.SHA,
    Cluster:  arrival.Cluster,
    Author:   resolveAuthor(suspectedPR, userLookup),
    Title:    fmt.Sprintf("Regression detected: %s %s", arrival.ServiceName, arrival.Version),
    Body:     renderForensicsDiff(arrival.Forensics),
    Links: map[string]string{
        "pr":      suspectedPR.URL,
        "results": resultStoreURL,
        "trace":   tempoTraceURL,
    },
    Tags: map[string]string{
        "env_classification": "staging_test",
        "newly_failed_count": fmt.Sprintf("%d", len(newlyFailed)),
    },
}
router.Notify(ctx, event) // routes to slack + lesson-capture (filtered) automatically
```

The consumer is now transport-agnostic. Adding Teams support tomorrow doesn't touch this code.

## Routing logic (where decisions happen)

The `Router` in `notify/router.go` implements the routing rules:

1. Look up `event.Type` in `routing` map → get list of always-fire transports
2. Evaluate `conditional_transports[].when` against the event → add matched transports
3. Evaluate `only_when` filters → drop the entire event if not matched
4. Build the actual `Notifier` (use `FanoutNotifier` if multiple transports)
5. Call `Notify(ctx, event)`

Conditions support a small DSL:

```yaml
when:
  author.is_automated_agent: true       # boolean filter
  severity: error                        # equality
  score: "<60"                           # comparison
  cluster: ["gcp", "az"]                 # any-of
  tags.env: "staging"                    # tag-equality
```

Keep the DSL **small**. If complex routing is needed, push it upstream into the producer's event construction (set richer tags). The Router doesn't try to be a full rules engine.

## Validation + dry-run

The `notify` package ships with:

- `notify validate <config.yaml>` — lints the config (referenced transport types exist, all GitHub handles in routing exist in `users:`, syntax of `when` clauses, etc.)
- `notify dry-run <event.json> --config <config.yaml>` — shows which transports the event would route to without dispatching. Used in CI on the qa-management repo.

Both ship as part of the package's `cmd/notify-cli/`. Run as part of qa-management's lint presubmit.

## Schema versioning

`notification-config.yaml` carries `schema_version: v1`. Router refuses unknown versions. Bump on backward-incompatible schema changes; consumers pin to a tag and bump via Renovate (same pattern as everything else in qa-management).

## Test strategy

`notify` package ships with unit tests + a `mocknotify.Notifier` fake for consumer integration tests. Pattern matches `mockk8s` in mqube-fat-controller — consumers don't need to wire real Slack to test "regression-detected fires the right event".

Phase 2.7 integration test goal: **arrivals-observer's regression-detection flow can be exercised end-to-end against the mock Notifier** without any Slack or automated-agent dependencies.

## Phase rollout

| Phase | What ships |
|---|---|
| **Session 0 spike** | Nothing — Notifier framework is not in the spike scope |
| **Phase 2.5 / 2.7 prep** | `leartech-go-common/notify/` package: types + Slack + Noop + Webhook + Fanout + Router. Initially used only by Phase 2.7 arrivals-observer. |
| **Phase 2.7 release** | `LessonCaptureNotifier` impl + author-filtering routing rules + automated-agent integration. Lesson-status `candidate` extension to automated-agent's schema (small extension on the agent side). |
| **Phase 3 / on-demand** | Teams, Email, PagerDuty impls as needed. `leartech-notification-service` standalone (centralized routing) only if cross-team coordination needs warrant it; the package-level Router is sufficient initially. |

Cost added vs the original "direct Slack webhook" plan: **~half day** for the framework + Slack impl in Phase 2.7. Same code in a different shape; no calendar impact on Phase 2.7's ~3-week budget.

## Why this is better than the standalone `leartech-notification-service` first

Building a notification service first would be heavier:

- Operational overhead — another long-running service to deploy, monitor, scale
- Inter-team coordination — service ownership, on-call, deprecation policy
- Premature abstraction — until multiple consumers actually need centralized routing, the centralized service is over-engineering

The `notify` package gives you 80% of the benefit (decoupling, testability, transport pluggability) at 20% of the cost. The standalone service is a **future option** when the consumer count justifies it; until then, each consumer carries its own thin Router instance reading the same shared config.

If/when the standalone service does ship, the consumer-side change is trivial: replace the local Router with an HTTP client that POSTs the Event to the service. The `Notifier` interface in consumers stays unchanged.

## Cross-references

- **`arrivals-observer.md`** — first major consumer; uses `arrivals.regression` + `arrivals.timeout` events with author-filtering for lesson capture
- **`gate.md`** — uses `gate.override` events (and could fire `gate.fail` for high-risk override audits)
- **`risk-assessor.md`** — uses `risk.high` events (currently muted but framework-ready)
- **`qa-management-repo.md`** — `notification-config.yaml` schema lives there
- **`~/leartech/automated-agent/gate/agent/lessons/`** — destination for `lesson-capture` transport; manual CLI also available for QA-analysis deep-dives
- **`~/leartech/hub/shared-rules/no-inline-python-in-tekton.md`** — the one rule that applies when shipping the package's CLI tools
