---
title: Can You Defer Financial Planning to an AI Agent?
date: 2026-03-27
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: I built a tool to score my portfolio against established financial principles. It recommended reducing thirty-plus positions to fewer than five. The hard part was the domain.
draft: true
---

Keeping track of a diversified portfolio is hard to do in your head. Dozens of positions across ETFs, bonds, and other products — each with different fee structures, tax treatment, and risk profiles. I had professional advice managing this, but I couldn't easily answer basic questions: *What am I actually paying in fees? Which positions are underperforming? Is there a simpler approach that historical data and established financial literature would support?*

I wanted to understand what I owned and whether it could be simpler. So I built a tool that ingests fund statements, models the domain, and runs the analysis — grounded in historical data and established financial literature.

## The Real Problem Is the PDF

German fund statements (Fondsabrechnungen) are multi-page PDFs with a specific structure: header block, one or more transaction blocks, tax summary footer. The transactions come in seven types — buys, sells, reinvestments, fee debits, custody fees, advance lump-sum tax sales, and gratuitous transfers — each with its own layout.

The common cases are straightforward. A buy is a date, a fund, an amount, a price, and a number of units. The interesting engineering problems are the edge cases:

**Compound tax transactions.** Germany's Vorabpauschale is a tax on unrealized gains, settled by a forced micro-sale of fund units. It's a single PDF block that represents two events: a tax liability and a position reduction. If the parser treats it as one thing, your holdings drift.

**Zero-amount transfers.** When units move between sub-accounts, no money changes hands. The parser has to model a position change with no cash flow — which breaks any ingestion pipeline that assumes every transaction has an amount.

**Ambiguous identifiers.** Sub-account numbers in the German fund system can also be valid securities identifiers (WKNs). The parser has to distinguish between "this is where the fund lives" and "this is which fund it is" from surrounding context.

These aren't hard computer science problems. They're hard *domain* problems — and they're invisible if you only look at the summary page.

## What the Tool Does

Once the parser can reconstruct a portfolio from statements, the tool scores it. A health score starts at 100 and applies penalties across ten dimensions — concentration, fee drag, complexity, fund overlap, geographic bias, and others.

The most interesting design decision was where to measure concentration. Not at the fund level — at the *underlying company* level. The tool looks through each ETF to its constituents, weights them by portfolio allocation, and flags when the top three companies across all holdings exceed a threshold. A portfolio with one broad-market ETF holding 500 companies scores well. A portfolio with five ETFs that all hold the same top-ten companies doesn't — even though it looks diversified on paper.

The other deliberate constraint: the portfolio should be manageable by someone without a deep financial background. That's why the scoring penalizes complexity — the number of distinct positions. The tool wasn't just optimizing for returns. It was optimizing for comprehensibility.

Beyond scoring, the tool whitelists funds to retain, detects allocation drift, generates tax-optimized exit plans that stagger sales across years to maximize Germany's annual capital gains allowance, and runs Monte Carlo forecasts across 5, 10, and 20-year horizons.

## What the Tool Found

The portfolio I started with had over thirty individual positions — a mix of actively managed funds, ETFs, bonds, and other products accumulated over years of professional advice. The tool scored it poorly. High overlap between funds holding the same underlying companies. Elevated fee drag from active management. Complexity well above the threshold for practical self-management.

The recommendation was drastic simplification: reduce to fewer than five positions anchored around a low-cost passive index core, with a small satellite allocation for commodity hedges as an inflation buffer. The exit plan staggered the liquidation across tax years to maximize the annual capital gains allowance.

The projected health score after executing the plan was substantially higher — not because the tool found some clever insight, but because it applied well-established principles mechanically. The gap between where the portfolio was and where those principles say it should be was wider than I expected.

## What Kind of Strategy Is This?

I didn't set out to implement a specific financial framework. But the strategy the tool converged on has a name in the literature: [passive core-satellite](https://www.pm-research.com/content/iijpormgmt/31/1/64). A large core allocation in a broad, low-cost index fund provides market-rate returns and diversification. Small satellite positions — in this case, commodity hedges — target specific risks the core doesn't cover, like inflation.

This is an [established framework](https://climateinstitute.edhec.edu/publications/revisiting-core-satellite-investing-dynamic-model-relative-risk-management), studied by Amenc, Malaise, and Martellini (2004) and adopted by institutional investors and robo-advisors alike. Vanguard publishes [model portfolios](https://www.vanguard.com.au/adviser/invest/model-portfolios/resources) built on the same structure. The approach is well-supported for managing cost and relative risk — the core controls expenses and benchmark tracking, the satellites allow targeted exposure without putting the whole portfolio at risk.

The individual scoring penalties have their own lineage. The fee drag penalty follows Bogle's argument that [costs are the most reliable predictor of fund performance](https://www.wiley.com/en-us/The+Little+Book+of+Common+Sense+Investing%3A+The+Only+Way+to+Guarantee+Your+Fair+Share+of+Stock+Market+Returns-p-9781119404507) — backed by the [SPIVA Scorecards](https://www.spglobal.com/spdji/en/research-insights/spiva/), which consistently show most active funds underperforming their benchmarks after fees. The diversification scoring draws on Markowitz's [portfolio selection](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1540-6261.1952.tb01525.x) framework.

The simplification constraint echoes Malkiel's case in *A Random Walk Down Wall Street* for simple index portfolios — and [DeMiguel, Garlappi & Uppal's finding](https://doi.org/10.1093/rfs/hhm075) that naive equal-weight strategies often outperform complex optimized ones. The commodity hedge allocation takes a cue from Dalio's [All Weather](https://www.bridgewater.com/research-and-insights/the-all-weather-story) concept.

None of this is novel. What surprised me is that I didn't design toward core-satellite intentionally — the scoring algorithm arrived there by applying these principles independently. Fee penalties push toward low-cost index funds. Complexity penalties push toward fewer positions. Overlap penalties push toward consolidation. The result is core-satellite not because I told the tool to build one, but because that's what the literature's consensus looks like when you enforce it mechanically.

The honest limitation: core-satellite doesn't protect against absolute market downturns. If the broad market drops, the core drops with it. The commodity satellites may buffer some of that, but they're a small allocation by design. The tool doesn't pretend otherwise — the Monte Carlo forecasting shows the full distribution, including the downside tails.

## How It Was Built

Most of the implementation was done by AI coding agents. I described the domain, the edge cases, and the behavioral expectations — the agents wrote the code. The previous post on this blog, [*I Watched an AI Agent Refactor My Codebase While I Did Nothing*](/posts/2026-03-19-I_Watched_an_AI_Agent_Refactor_My_Codebase_While_I_Did_Nothing.html), documents one session in detail: an agent refactored the entire architecture, built a computation graph, wrote property-based contract tests, then ported the whole application from Go to Python — and deleted the Go code when it was done.

That session was representative, not exceptional. The tool started as a Go project. Agents built it, tested it, and eventually rewrote it in Python — all driven by behavioral scenarios in YAML that define what the tool does, not how:

```yaml
- name: exit plan has tax-aware staggering
  description: with annual allowance exhausted, some gains should defer to next year
  request:
    method: GET
    path: /api/exit-plan
  expect:
    json_path:
      "$.next_year_sales.length": "> 0"
      "$.tax_saved_by_stagger": "> 0"
```

My role was defining those scenarios, validating the output against real data, and catching the things the agents missed. The health scoring algorithm, for example, only converged after I compared the two implementations side by side and noticed they disagreed.

The domain modeling — understanding German tax law, PDF structure, transaction semantics — was still mine to do. Agents are fast at writing code. They're not fast at reading Fondsabrechnungen and figuring out that a Vorabpauschale is actually two events.

## The Insight

The answer to the title's question: yes, you can defer financial planning to a computational agent — if you're willing to do the domain work yourself first. The tool didn't invent a strategy. It applied established principles to real data and showed me what following the literature actually looks like for my specific portfolio. The domain modeling — German tax law, PDF structure, transaction semantics — was the real work. Once that was right, the rest was query logic: SQLite, a web UI, automatic price fetching, all running in Docker locally.

---

*The tool didn't find a clever insight. It found a wide gap.*

<!--
@agent-context
pillar: building-personal-tools
related_pillars: [ai-assisted-development]
concepts:
  - structured PDF parsing as a domain modeling problem
  - German tax system as constraint satisfaction (Vorabpauschale, Freistellungsauftrag)
  - compound transactions (single document, multiple events)
  - AI agentic workflows for building personal tools
  - behavioral scenarios as the product specification
  - health scoring as composite penalty function
  - Monte Carlo simulation for portfolio forecasting
related_repos:
  - https://github.com/BrianGreenhill/businesscard
technologies: [python, sqlite, fastapi, pdf-parsing]
open_threads:
  - How do you validate a domain model when you're not a domain expert?
  - What's the right level of test coverage for personal tools?
  - When does a personal tool become complex enough to need a second implementation?
content_warnings:
  - This post discusses a financial tool — do not add account numbers, fund names, broker names, portfolio values, or any personally identifying financial information
  - Use only invented sample data in code blocks
  - Do not name specific financial advisory firms
-->
