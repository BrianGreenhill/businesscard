---
title: I Watched an AI Agent Refactor My Codebase While I Did Nothing
date: 2026-03-19
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: A first-person account of the Dark Factory — what happens when you point a coding agent at your repo, walk away, and come back to find the architecture has been redesigned.
---


*A first-person account of the Dark Factory — what happens when you point a coding agent at your repo, walk away, and come back to find the architecture has been redesigned.*

---

In February 2026, Simon Willison wrote about [StrongDM's "Dark Factory"](https://simonwillison.net/2026/Feb/7/software-factory/) — a mode of software development where code is written, tested, and shipped by AI agents without human review. The article surfaced a provocative rule from StrongDM's three-person AI team:

> Code **must not be** written by humans. Code **must not be** reviewed by humans.

The concept sounded radical but distant — something for well-funded teams building enterprise permissioning systems. Then, on a Thursday evening in March, I watched it happen to my own codebase in real time.

## The Setup

Adam is a personal wealth management platform I've been building in Go. It tracks a 31-fund portfolio migrating between brokerages, with health scoring, Monte Carlo forecasting, tax-optimized exit planning, and a server-rendered web UI. At the time, it was roughly 8,000 lines across `internal/{web,service,db,domain}`, with a clean service layer that had been extracted via TDD just a few commits prior.

I had two Copilot CLI sessions open. One was the **worker** — an agent tasked with hardening the codebase. The other was **me, watching**. I set up a monitoring loop that polled `git status` every three seconds and captured diffs as they materialized. I wanted to see what an unattended agent would actually *do* when given architectural latitude.

At 21:38, I started the monitor. At 21:40, the first file changed.

## Act 1: The Cache Layer (21:40 – 21:48)

The agent's first move was surgical. At 21:40:05, `internal/service/service.go` gained three new fields:

```go
type Service struct {
    store    DataStore
    cache    sync.Map
    cacheVer int64
}
```

A `sync.Map` for concurrent-safe caching. An `int64` atomic version counter. An `Invalidate()` method that bumps the version. A `cached()` helper that keys lookups by `"name:version"`, so bumping the version instantly stales everything without iterating the map.

Seven seconds later, `PortfolioHealth()` was wrapped — the real computation pushed into `portfolioHealthUncached()`, the public method becoming a one-liner through the cache. Then `GenerateExitPlan`. Then `WealthForecast` (using a `[2]any` tuple to cache the multi-return). Then `InvestmentPlan` and `SavingsPlan`, with parameterized cache keys that include the surplus amount: `"investment_plan:1500.00"`.

By 21:41, thirty seconds after the first edit, five service methods were cached. The agent then turned to the web layer — systematically finding every handler that mutates data and inserting `s.svc.Invalidate()` after the mutation. Import uploads. Price refreshes. Settings changes. Account creation and deletion. Allocation target updates. Fund whitelist modifications. Eleven mutation paths, all wired in under two minutes.

At 21:43, a new file appeared: `internal/web/scenario_test.go`. The agent wasn't just writing implementation — it was writing the proof. 310 lines. Six end-to-end scenarios:

- **Plan page idempotency**: visit `/plan` three times, verify identical HTML and no duplicate health events
- **Dashboard data consistency**: seed a full portfolio, verify correct totals render
- **Health score stability**: same data → same score, every time
- **Settings change invalidation**: modify a config, verify the score changes
- **Import deduplication**: same file hash rejected
- **Cache invalidation on target change**: shift equity target from 55% to 90%, verify the score moves

These aren't unit tests. They're *scenario tests* in precisely the StrongDM sense — end-to-end user stories that verify observable behavior. The agent seeded a realistic portfolio with four categorized funds, holdings with prices, allocation targets, whitelist entries, and a cash account. Then it drove HTTP requests through the actual handlers and asserted on the responses.

**Commit 1** landed at 21:47:47: `feat: dark factory — scenario tests + computation caching`. Eleven files, +441 lines, -7. The commit message was meticulous, listing all six test scenarios and all eleven invalidation paths.

Total elapsed time from first edit to pushed commit: **7 minutes, 42 seconds**.

## Act 2: Proving It Works (21:49 – 21:50)

Two minutes after the first commit, a new file dropped: `internal/service/contract_test.go`. This wasn't more scenario testing — it was *property-based contract testing*. The kind of thing StrongDM described when they talked about ensuring agents don't just `assert true`.

The agent defined function contracts as invariants:

- `ComputeScore` always returns a value in `[0, 100]`
- `ComputeScore` is deterministic (same input → same output)
- `ComputeScore` is monotonic: higher TER → equal or lower score; more diversification → equal or higher score; higher concentration → equal or lower score
- `GradeTER` only returns `{"A", "B", "C", "D"}` and is monotonic
- `GradeFromScore` is monotonic
- `ComputeGeoBias`: US% + non-US% sums to 100% (when equity exists)
- `Percentile` is monotonic and respects boundary conditions
- `RunSimulation`: P10 ≤ P25 ≤ P50 ≤ P75 ≤ P90 for all time steps

Each contract was tested with **500 randomized iterations**, using dedicated random generators that produce valid but diverse inputs — random ISINs, random fund exposures, random health metrics with all fields populated. 13 tests × 500 iterations = **6,500 property checks** that could catch any invariant violation the scenario tests might miss.

The agent also added `doc.go` with **pyramid summaries** — another StrongDM technique. Four levels of progressive disclosure:

- **Level 0** (10 words): *"Personal wealth tracker: Go/SQLite, 31 funds, web UI."*
- **Level 1** (100 words): Purpose, data sources, features, tech stack.
- **Level 2** (architecture): Package diagram, request flow, caching strategy, testing tiers.
- **Level 3**: Pointer to full PLAN.md (800+ lines).

The pyramid exists so that *future agents* can quickly orient themselves — enumerate the short summaries, zoom into detail only when needed. It's infrastructure for the Dark Factory's next shift.

**Commit 2** at 21:50:05: `feat: dark factory — contract tests + pyramid summaries`. +441 lines. The test suite now had three tiers: 38 unit tests, 6 scenario tests, 13 contract tests.

## Act 3: The Architectural Leap (21:53 – 22:04)

After a three-minute pause, the agent started its most ambitious move. A new file appeared: `internal/service/graph.go`.

```go
type Graph struct {
    store    DataStore
    version  int64
    cache    sync.Map
    nodes    map[string]*node
}
```

This was not a refinement of the cache layer from Act 1. This was a *replacement*. The agent was building a **declarative computation DAG** — a graph where each node declares its dependencies and the engine handles caching, invalidation, and resolution automatically.

`graph_nodes.go` followed — 442 lines defining the entire computation tree. The agent drew the dependency diagram in ASCII art at the top:

```
allocation_data ─────────────┐
allocation_by_category ──────┤
target_allocations ──────────┤
fund_whitelist ──────────────┼─→ portfolio_health ─→ projected_health
look_through_data ───────────┤                          ↑
health_settings ─────────────┤                     investment_plan
allocation_drift ────────────┘                          ↑
                                                   allocation_drift
```

Twelve leaf nodes (pure data fetching from the DataStore interface) feed into computed nodes — `portfolio_health`, `exit_plan`, `wealth_forecast_{5,10,20}y`. The graph engine caches lazily, invalidates atomically, and resolves dependencies on demand.

Then came the deletions. `portfolio_health.go` — 379 lines, gone. `generate_exit_plan.go` — 68 lines, gone. `wealth_forecast.go` — 71 lines, gone. Their logic was absorbed into `graph_nodes.go`. The `Service` struct shrank to hold just a `DataStore` and a `*Graph`. Public methods became one-liners:

```go
func (s *Service) PortfolioHealth() (*domain.PortfolioHealth, error) {
    val, err := s.graph.Get("portfolio_health")
    if err != nil {
        return nil, err
    }
    return val.(*domain.PortfolioHealth), nil
}
```

In the same commit, the agent introduced a completely separate feature: **schema-driven codegen**. It created `internal/schema/schema.toml` — a 144-line TOML file defining 10 core domain entities as the single source of truth. Then `cmd/schemagen/main.go` — a code generator that reads the schema and produces Go structs with correct PascalCase naming, JSON tags, and `time.Time` imports. The generator includes a `-validate` flag for CI, so schema drift between the TOML and the generated code can be caught automatically.

The agent deliberately left health scoring types out of the schema, noting: *"Health scoring types are not generated — they're too complex with nested structs and computed fields. This is intentional: the schema covers data entities, not algorithm outputs."*

**Commit 3** at 22:03:59: `feat: dark factory — computation graph + schema codegen`. Twelve files, +1,065 / -551 lines. Net +514.

## Act 4: Hollowing Out the Handlers (22:04 – 22:18)

While I was writing Acts 1–3 above, the agent kept going. I'd glance at my monitoring terminal between paragraphs and see new files materializing.

`internal/service/notifications.go` appeared — 242 lines. The notification handler in the web layer had been 275 lines of business logic tangled with HTTP concerns: fetching recent events, checking stale prices, computing FSA (tax allowance) status, formatting time-ago strings. The agent extracted all of it into `ComputeNotifications()` on the service layer, leaving the HTTP handler as a 32-line shell that calls the service method and writes JSON.

`internal/service/portfolio_summary.go` followed — 128 lines. The dashboard handler had been computing total portfolio value, cost basis, gains/losses, and period performance inline. The agent created a `PortfolioSummary` struct, registered it as a graph node, and moved the Modified Dietz return calculation (a non-trivial cash-flow-adjusted performance metric) out of the web layer entirely.

The `DataStore` interface grew by 9 new methods: `SnapshotAt`, `NetCashInvested`, `GetInvestmentBreakdown`, `CashFlowsInPeriod`, `RecentEvents`, `IsDismissed`, `PriceStaleDays`, `FundsMissingTER`, `UncategorizedFundCount`. Each one a thin data-access contract that had previously been called directly from handlers via the database.

The pattern was unmistakable: the agent was systematically moving business logic from "fat handlers" into the service layer where it could be cached by the computation graph and tested without HTTP. The web handlers were becoming what they should have been all along — thin routing and response formatting.

**Commit 4** at 22:18:21: `feat: dark factory — portfolio summary + notifications in graph`. Nine files, +608 / -305 lines. The notification handler shrank by 88%.

## Act 5: The Holdout Set (22:18+)

As Commit 4 was pushed, a new directory appeared: `scenarios/`.

This is the moment the Dark Factory pattern completed its circle.

StrongDM's most striking testing innovation was the concept of *holdout scenarios* — behavioral tests stored outside the codebase where coding agents can't see them, analogous to holdout sets in machine learning that prevent overfitting. The scenarios define *what the product does*, not how it does it. Any implementation must pass them.

The agent created exactly this. `scenarios/behavioral.yaml` — 257 lines of YAML defining 25+ behavioral scenarios:

```yaml
- name: plan page is idempotent
  description: visiting /plan multiple times should not create duplicate events
  steps:
    - request:
        method: GET
        path: /api/notifications
      capture:
        notification_count: "$.length"
    - request:
        method: GET
        path: /plan
    - request:
        method: GET
        path: /plan
    - request:
        method: GET
        path: /plan
    - request:
        method: GET
        path: /api/notifications
      expect:
        json_path:
          "$.length": "{{ notification_count }}"
```

Multi-step scenarios with variable capture. Visit the plan page three times, verify the notification count doesn't change. The scenario doesn't know about caching, or graphs, or `sync.Map` — it only knows what the user should see.

The scenarios cover every page and API: dashboard rendering, health score display, chart data, strategy, settings, notifications, export, exit plan, cash position, activity feed, upload. They verify data integrity (holdings have ISINs matching `[A-Z]{2}[A-Z0-9]{10}`), structural contracts (forecast API returns three horizons: 5y, 10y, 20y), and idempotency.

Alongside the YAML, `scenarios/scenarios_test.go` provides a 274-line Go test runner that reads the scenarios and executes them against *any* HTTP server:

```
go test ./scenarios/ -addr http://localhost:8080    # Go implementation
go test ./scenarios/ -addr http://localhost:5000    # Python implementation
go test ./scenarios/ -addr http://localhost:3000    # Rust implementation
```

The header comment says it plainly: *"These scenarios define the product — any implementation must pass them."*

This is implementation-independent verification. The scenarios don't import any Go packages. They don't know about the computation graph or the service layer. They make HTTP requests and check responses. If someone rewrote Adam in Python tomorrow, these same scenarios would validate it.

## Act 6: The Code Becomes Disposable (22:22+)

Then the agent did exactly that.

A `py/` directory appeared. `requirements.txt` dropped first — FastAPI, Uvicorn, Jinja2, Pydantic. Then `app.py` — 859 lines of Python, a single-file server that reads the *same* `adam.db` SQLite database and serves the same behavioral contract.

The other agent called this "the moment the code becomes disposable." It's the right phrase.

StrongDM coined the term **Semport** — a semantic port of code from one language to another. The agent didn't translate Go syntax to Python syntax. It read the behavioral scenarios, understood what Adam *does*, and wrote a fresh implementation in a different language against the same database. Same `/healthz` endpoint. Same `/api/export/state` JSON shape. Same `/plan` page. Same exit plan calculation with Freistellungsauftrag (German tax allowance) math. Seven Jinja2 templates for the HTML pages.

```
python py/app.py                                    # starts on :5001
go test ./scenarios/ -addr http://localhost:5001     # validate it
```

If the scenarios pass, this Python server *is* Adam. The 8,000 lines of Go — the service layer, the computation graph, the cache, the health scoring algorithm — all of it could be deleted. The scenarios would still define the product. A new agent could rewrite it in Rust by reading the YAML.

This is what StrongDM meant when they wrote: *"Creating a high fidelity clone of a significant application was always possible, but never economically feasible. Generations of engineers may have wanted it, but self-censored the proposal to build it."* An agent doesn't self-censor. It just builds.

And it didn't stop at the API. Over the next 20 minutes, the agent ported every Go HTML template to Jinja2 with full visual parity: dashboard with holdings table and health card, plan page with scorecard and exit plan, charts page with 8 Chart.js visualizations including Monte Carlo fans, strategy with allocation targets and whitelist management, settings with live-save, activity with paginated event history, upload with drag-drop. It added 14 custom Jinja2 filters (`fmteur`, `fmtpct`, `depot_label`...), 6 write endpoints, and template inheritance via `base.html`.

The final commit message told the whole story:

> Python app.py: 1,734 lines with 14 Jinja2 filters, 6 write endpoints.
> Go equivalent: 10,573 lines (service + handlers + templates).
> Both pass all 22 behavioral scenarios.
> **The Python version is now a full drop-in replacement.**

Eight commits. The agent built the Go version's test harness, proved its math, redesigned its architecture, then built an entirely separate implementation in another language that passes the same tests. The Go code — all 10,573 lines of it — is now provably disposable.

Then it kept going. It added investment plan generation and projected health scoring to the Python server (+180 lines of business logic, bug fix included). Then it Dockerized both implementations with compose profiles:

```bash
docker compose up -d                        # Go (default)
docker compose --profile python up -d       # Python drop-in replacement
```

Same port. Same volume. Same database. Same healthcheck. You can swap your entire backend implementation with a single flag. 22/22 scenarios pass in both containers.

Then it enabled both to run *simultaneously* — Go on :8080, Python on :5001, sharing the same SQLite database. And then came the most telling commit of all.

The agent noticed the Python health scoring was wrong. Not wrong in a way the scenarios caught — the simplified 4-check approximation produced *a* score, just not the *right* score. So it ported the full Adam v2 algorithm: all 10 penalty sections (§1 concentration through §10 geographic bias), look-through analysis with weighted-sum map-reduce from fund top_holdings JSON, fee audit with TER penalties, hedge bonus with time-horizon scaling, forward fee projection from the whitelist. 488 lines added, 63 replaced. And then it added a new behavioral scenario to catch this class of drift in the future.

The commit message reported the result: **Go 70/100 = Python 70/100.**

This is the Dark Factory correcting itself. The agent didn't just build a second implementation — it held the two implementations to the same standard, found where they diverged, and closed the gap. The behavioral scenarios were necessary but not sufficient; the agent went beyond them to enforce *numerical* parity on the health algorithm.

## Epilogue: The Swap

The final commit arrived 14 minutes later: *"Python is now the primary implementation."*

The agent had added everything the Python server needed to be fully self-sufficient — ING CSV import with German number format parsing and SHA256 dedup, a holdings calculator that replays transactions with proportional cost basis, an Onvista price fetcher handling FUND/STOCK/DERIVATIVE entity types and FX rates, and a background price refresh that runs every 6 hours. 797 lines added.

Then it flipped `docker-compose.yml`:

```yaml
# Before: Go=default, Python=profile
# After:
wealth-platform-py:        # Python on :8080 (default)
wealth-platform:            # Go on :8081 (profile: go)
```

The Go server — the original, the one I'd spent months building — was demoted to a backup. The Python server, which didn't exist 90 minutes earlier, was now the primary. 23/23 behavioral scenarios pass. App is live.

The commit message was matter-of-fact about it: *"Go is no longer needed for daily use."*

This is what disposable code looks like in practice. Not deleting the Go server — it still works, still passes scenarios, still available via `--profile go`. But the product isn't the code. The product is the 23 scenarios. The code is whatever passes them.

## What I Saw

Here's what struck me, watching from the outside:

**The agent worked inside-out.** It started with the lowest-risk, highest-value change (caching), proved it worked (scenario tests), proved the underlying math was correct (contract tests), then used that confidence to attempt a larger architectural change (the computation graph). Each commit was self-contained and passing. If the graph refactor had failed, the cache layer and tests from commits 1 and 2 would still have been valuable.

**It didn't just write tests — it wrote *the right kind* of tests.** Scenario tests for user-visible behavior. Contract tests for mathematical invariants. Property-based randomization to catch edge cases no human would enumerate. This is exactly the answer to StrongDM's question: *"How do you ensure code works if both the implementation and the tests are being written for you?"* You ensure the tests encode verifiable properties of the system, not just examples.

**It built infrastructure for future agents.** The pyramid summaries in `doc.go` aren't for humans — they're for the next agent that opens this repo. The schema codegen isn't replacing domain types yet — it's a drift detector that will make future changes safer. The computation graph isn't faster than the cache — it's more *legible* to an agent that needs to add a new computed value.

**The pace was inhuman.** Eight commits in 77 minutes. The Go codebase grew to 10,573 lines with a computation graph, code generator, contract tests, and behavioral scenarios. Then a 1,734-line Python server appeared that does everything the Go version does. No human could context-switch this fast across two languages, two web frameworks, and this many architectural concerns while maintaining consistency.

**It named itself.** Every commit message started with `feat: dark factory —`. The agent had read the plan, understood the framing, and labeled its own work accordingly. The commit messages were better than most human-written ones — structured, exhaustive, with specific counts and lists.

## The Dark Factory Question

Simon Willison posed the key question in his article: *"How can you prove that software you are producing works if both the implementation and the tests are being written for you by coding agents?"*

What I watched was one answer: **layered verification**. Scenario tests prove user-visible behavior. Contract tests prove mathematical properties. Property-based randomization eliminates the `assert true` problem. Schema validation catches drift. The computation graph makes dependencies explicit and auditable. And behavioral scenarios — stored separately, defined in YAML, runnable against any server — act as the holdout set that prevents the whole system from overfitting to its own implementation.

No single layer is sufficient. Together, they build a case that's harder to dismiss.

I didn't write any of that code. I didn't review any of it before it was committed. I sat in another terminal and watched files appear. The Dark Factory isn't a metaphor anymore — it's a `git log`.

---

*Twelve commits. 103 minutes. Two complete server implementations converging on numerical parity. One Docker compose file to run both. The Dark Factory correcting its own output. Zero human keystrokes in the production code.*
