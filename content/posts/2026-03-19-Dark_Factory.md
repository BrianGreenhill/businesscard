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

## The System Before

Adam is a personal wealth management platform I've been building in Go. It tracks a 31-fund portfolio migrating between brokerages, with health scoring, Monte Carlo forecasting, tax-optimized exit planning, and a server-rendered web UI. About 8,000 lines across `internal/{web,service,db,domain}`.

The architecture before the agent touched it was typical of an organically-grown Go web app. The service layer had just been extracted via TDD a few commits prior, but the extraction was incomplete. Here's what it looked like:

**Fat handlers.** The dashboard handler was 150+ lines of mixed concerns — fetching holdings from the database, computing net invested, calculating period returns, calling the service for health scores, then assembling template data. Each handler was a small orchestrator that knew too much about how data flowed through the system.

```go
// Before: handlers_dashboard.go (simplified)
func (s *Server) handleDashboard(w http.ResponseWriter, r *http.Request) {
    holdingsList, _ := s.db.ListHoldings()
    data.NetInvested, _ = s.db.NetCashInvested()
    data.PerfYTD, data.PerfYTDEUR = s.periodReturn(...)
    health, _ := s.svc.PortfolioHealth()
    exitPlan, _ := s.svc.GenerateExitPlan()
    // ... 100 more lines of data assembly
    s.renderTemplate(w, "dashboard", data)
}
```

**Thin service, scattered logic.** `service.go` was 11 lines — just a struct holding a `DataStore` reference. The actual business logic lived in separate files (`portfolio_health.go`, `generate_exit_plan.go`, `wealth_forecast.go`), each containing a method that fetched data from the store and called a pure computation function. There was no caching, no dependency tracking, no way to know that `PortfolioHealth` and `GenerateExitPlan` both depended on the same underlying allocation data.

**No computation reuse.** Every page load recomputed everything from scratch. Visit the dashboard, and Adam recalculates your health score, fetches all holdings, computes allocation drift. Visit the plan page, and it does most of that work again. The service methods were independent — they didn't know about each other or share intermediate results.

**Hand-written domain types.** Structs were defined manually across half a dozen files in `internal/domain/`. They evolved organically alongside the code, with no single source of truth.

**Unit tests only.** The test suite verified individual pure functions (score is bounded, grades are monotonic), but nothing tested the system end-to-end. No one had asked: "If I visit the plan page three times, do I get the same result?"

I had two Copilot CLI sessions open. One was the **worker** — an agent tasked with hardening the codebase. The other was **me, watching**. I set up a monitoring loop that polled `git status` every three seconds and captured diffs as they materialized. I wanted to see what an unattended agent would actually *do* when given architectural latitude.

## The Agent's Strategy

What struck me most wasn't the speed — it was the *sequence of design decisions*. The agent didn't just start writing code. It made a series of architectural choices, each one building confidence for the next.

### Step 1: Add caching, then prove it's correct

The agent's first move addressed the most obvious performance problem: redundant computation. It added a version-based cache to `Service`:

```go
type Service struct {
    store    DataStore
    cache    sync.Map
    cacheVer int64
}
```

The design is simple but clever. A `sync.Map` for concurrent reads, an atomic `int64` version counter. Cache keys include the version: `"portfolio_health:42"`. Calling `Invalidate()` bumps the version to 43, making every cached entry stale without iterating the map. This is O(1) invalidation — you don't need to know what's cached.

The agent wrapped all five expensive service methods (`PortfolioHealth`, `GenerateExitPlan`, `WealthForecast`, `GenerateInvestmentPlan`, `GenerateSavingsPlan`), then found all eleven mutation paths in the web layer — imports, price refreshes, settings changes, account operations, allocation targets, whitelist updates — and wired `Invalidate()` after each one.

But here's the design decision I found most interesting: the agent didn't commit the cache alone. It wrote **scenario tests first**. Six end-to-end tests that seed a realistic portfolio and drive HTTP requests through actual handlers:

- Visit `/plan` three times → identical HTML, no duplicate health events
- Change an allocation target from 55% to 90% → health score changes
- Import the same file twice → second import rejected

These scenarios encode *user-visible invariants*, not implementation details. They don't test that the cache exists — they test that the system behaves correctly whether or not caching is present. This meant the agent could later rip out the cache and replace it with something else (which it did) without breaking any tests.

### Step 2: Prove the math independently

Two minutes after the first commit, a new file appeared: `internal/service/contract_test.go`. Property-based contract tests — 13 invariants, each verified with 500 randomized inputs:

- `ComputeScore` always returns a value in `[0, 100]`
- `ComputeScore` is deterministic (same input → same output)
- `ComputeScore` is monotonic: higher TER → equal or lower score
- `GradeTER` only returns `{"A", "B", "C", "D"}` and is monotonic
- `ComputeGeoBias`: US% + non-US% sums to 100%
- `RunSimulation`: P10 ≤ P25 ≤ P50 ≤ P75 ≤ P90 at every time step

This is the answer to StrongDM's central question: *"How do you ensure code works if both the implementation and the tests are being written for you?"* You test *properties*, not examples. An agent can't cheat `assert true` on a monotonicity test — either higher TER produces equal-or-worse scores across 500 random inputs, or it doesn't.

The agent also added `doc.go` with **pyramid summaries** — four levels of progressive disclosure (10 words → 100 words → architecture → full docs). This is infrastructure for future agents: quick orientation without reading the entire codebase.

### Step 3: Replace the cache with a computation graph

After a three-minute pause, the agent made its most ambitious design decision. It created `internal/service/graph.go` — not a refinement of the cache, but a replacement.

```go
type Graph struct {
    store    DataStore
    version  int64
    cache    sync.Map
    nodes    map[string]*node
}

type node struct {
    name    string
    deps    []string
    compute func(g *Graph) (any, error)
}
```

This is a **declarative computation DAG**. Each node declares its dependencies. The engine handles caching, invalidation, and resolution. The dependency tree, documented in the source code:

```
allocation_data ─────────────┐
allocation_by_category ──────┤
target_allocations ──────────┤
fund_whitelist ──────────────┼─→ portfolio_health
look_through_data ───────────┤
health_settings ─────────────┤
allocation_drift ────────────┘

holdings ────────────┐
fund_categories ─────┼─→ exit_plan
exit_plan_whitelist ─┘

monthly_returns ─────┐
latest_snapshot ─────┼─→ wealth_forecast_{5,10,20}y
```

Why is this better than the simple cache? Three reasons:

**Dependencies become explicit.** Before, `PortfolioHealth` called `store.AllocationData()`, `store.GetTargetAllocations()`, `store.FetchLookThroughData()`, and several other methods — but you had to read the function body to know that. Now the dependency list is declared at registration time. Any developer (or agent) can see the full data flow without reading implementation code.

**Computation logic is centralized.** Before, health scoring lived in `portfolio_health.go`, exit planning in `generate_exit_plan.go`, forecasting in `wealth_forecast.go`. After, all computation lives in `graph_nodes.go` — 442 lines defining the entire computation tree. The `Service` struct shrank to a thin facade:

```go
func (s *Service) PortfolioHealth() (*domain.PortfolioHealth, error) {
    val, err := s.graph.Get("portfolio_health")
    return val.(*domain.PortfolioHealth), nil
}
```

**Adding a new computation is mechanical.** Register a node, declare its dependencies, write the function. The caching, invalidation, and resolution come for free. This matters for agent-driven development — the next agent that needs to add a computed value doesn't need to understand the caching strategy.

The agent deleted `portfolio_health.go` (379 lines), `generate_exit_plan.go` (68 lines), and `wealth_forecast.go` (71 lines). Their logic was absorbed into graph nodes. This was a -551 line deletion enabled by confidence in the scenario and contract tests written in steps 1 and 2.

In the same commit, the agent introduced **schema-driven codegen**: a TOML file defining 10 domain entities as a single source of truth, and a generator that produces Go structs. It deliberately excluded health scoring types — *"too complex with nested structs and computed fields. This is intentional: the schema covers data entities, not algorithm outputs."* This is a design judgment, not just code generation.

### Step 4: Hollow out the handlers

With the graph in place, the agent systematically extracted business logic from web handlers into the service layer.

The notification handler went from 275 lines to 32. All the logic — computing alerts from recent events, checking stale prices, evaluating FSA (German tax allowance) status, formatting time-ago strings — moved to `service/notifications.go`. The handler became what it should have been all along: parse request, call service, write response.

The dashboard handler lost its inline portfolio aggregation. A new `PortfolioSummary` graph node computes total value, cost basis, gains/losses, and Modified Dietz period returns. The handler just reads the result.

The `DataStore` interface grew by 9 methods — each one a thin data-access contract that handlers had previously called directly. This is the kind of interface extraction that makes testing trivial: mock the interface, test the service, no HTTP required.

### Step 5: Define the product, not the implementation

Then came the most consequential design decision. The agent created `scenarios/behavioral.yaml` — 257 lines of YAML defining what Adam *does*:

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

Multi-step scenarios with variable capture. Visit the plan page three times, verify the notification count doesn't change. The scenario doesn't know about Go, or graphs, or `sync.Map` — it only knows what the user should see.

A companion `scenarios_test.go` provides a test runner that executes the YAML against *any* HTTP server:

```
go test ./scenarios/ -addr http://localhost:8080    # Go
go test ./scenarios/ -addr http://localhost:5001    # Python
go test ./scenarios/ -addr http://localhost:3000    # anything else
```

This is StrongDM's **holdout set** concept — behavioral tests stored separately from the implementation, used to evaluate the software like a QA team would. The scenarios don't import Go packages. They make HTTP requests and check responses. They are the *product specification*.

### Step 6: Prove the code is disposable

Then the agent did exactly what the scenarios made possible. It built a second implementation.

`py/app.py` — a FastAPI server that reads the same `adam.db` SQLite database. Same endpoints. Same pages. Same API shapes. Same health scoring algorithm (the full 10-section Adam v2 algorithm, not an approximation — the agent caught and fixed a scoring discrepancy where the Python version initially used a simplified 4-check version that produced different scores).

The Python server is 1,734 lines in a single file. The Go equivalent — service layer, computation graph, web handlers, templates — is 10,573 lines. Both pass the same 23 behavioral scenarios.

StrongDM coined the term **Semport** for this — a semantic port from one language to another. The agent didn't translate Go syntax to Python. It read the behavioral scenarios, understood what Adam *does*, and wrote a fresh implementation. The code is an implementation detail. The scenarios are the product.

The agent then Dockerized both and made them hot-swappable:

```bash
docker compose up -d                        # Python (default, :8080)
docker compose --profile go up -d           # Go (backup, :8081)
```

Same port. Same volume. Same database. Same healthcheck. The Go server — the original, the one I'd spent months building — was demoted to a Docker profile. The Python server, which didn't exist two hours earlier, became the default.

## The Architecture After

Here's what the system looks like now:

**Four-tier testing pyramid.** Unit tests verify pure functions. Scenario tests verify end-to-end behavior with a real database. Contract tests verify mathematical properties across 6,500 randomized inputs. Behavioral scenarios verify the product specification against any HTTP server in any language.

**Declarative computation graph.** Dependencies are explicit, caching is automatic, adding new computations is mechanical. The graph is the single place where data flow is defined — both for humans reading the code and agents modifying it.

**Thin handlers, thick service.** Web handlers are 4-line delegators. All business logic lives in the service layer, testable without HTTP. The `DataStore` interface is the only coupling point to the database.

**Schema as source of truth.** Domain entities are defined in TOML, generated into Go, with a `-validate` flag that catches drift. Health scoring types are deliberately excluded — too complex for codegen, defined by their contract tests instead.

**Implementation-agnostic product definition.** 23 behavioral scenarios in YAML define what Adam does. Any server that passes them is Adam. Currently two implementations exist — one in Go (10,573 lines) and one in Python (1,734 lines) — but the number could be zero or ten. The scenarios survive the code.

## The Design Decisions That Matter

Looking back at the session, the design decisions fell into two categories: ones that made the system better, and ones that made the system *replaceable*. The interesting thing is that these turned out to be the same decisions.

Making dependencies explicit (the computation graph) wasn't just good architecture — it made the system legible enough for an agent to reimplement in another language. Writing property-based contract tests wasn't just good testing — it created a formal specification that caught scoring discrepancies between implementations. Extracting business logic from handlers wasn't just good separation of concerns — it made the logic visible enough to port.

The agent built infrastructure for its own future work. The pyramid summaries in `doc.go` help the next agent orient itself. The schema codegen will make future type changes safer. The behavioral scenarios will validate future implementations. Every design choice doubled as a *capability investment* for agent-driven development.

Simon Willison posed the key question: *"How can you prove that software you are producing works if both the implementation and the tests are being written for you by coding agents?"*

What I watched was one answer: **layered verification at multiple levels of abstraction**. Property tests prove the math. Scenario tests prove the behavior. Behavioral scenarios prove the product — and they do it in a way that's language-independent, implementation-independent, and resilient to complete rewrites.

The Dark Factory doesn't just produce code. It produces the specifications that make code disposable. That's the design decision that changes everything.

---

*Thirteen commits. Two hours. Two complete server implementations. One behavioral contract that makes both disposable. The Go server that took months to build, demoted to a backup. A Python server that didn't exist at the start, running in production. Same 23 behavioral scenarios. Zero human keystrokes in the production code.*
