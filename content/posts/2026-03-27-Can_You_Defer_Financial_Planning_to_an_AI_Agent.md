---
title: Can You Defer Financial Planning to an AI Agent?
date: 2026-03-27
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: I built a tool to score my portfolio against established financial principles. It recommended reducing thirty-plus positions to fewer than five. The hard part was the domain.
draft: true
---

Keeping track of a diversified portfolio is hard to do in your head. Dozens of positions across ETFs, bonds, and other products — each with different fee structures, tax treatment, and risk profiles. I had professional advice managing this, but I couldn't easily answer basic questions: *What am I actually paying in fees? Which positions are underperforming? Is there a simpler approach that historical data and established financial literature would support?*

I wanted to understand what I owned and whether it could be simpler. So I built a tool that ingests fund statements, models the domain, and runs the analysis — grounded in historical data, established financial literature, and hedge fund strategies I'd been reading about.

## The Real Problem Is the PDF

German fund statements (Fondsabrechnungen) are multi-page PDFs with a specific structure: header block, one or more transaction blocks, tax summary footer. The transactions come in seven types — buys, sells, reinvestments, fee debits, custody fees, advance lump-sum tax sales, and gratuitous transfers — each with its own layout.

The common cases are straightforward. A buy is a date, a fund, an amount, a price, and a number of units. The interesting engineering problems are the edge cases:

**Compound tax transactions.** Germany's Vorabpauschale is a tax on unrealized gains, settled by a forced micro-sale of fund units. It's a single PDF block that represents two events: a tax liability and a position reduction. If the parser treats it as one thing, your holdings drift.

**Zero-amount transfers.** When units move between sub-accounts, no money changes hands. The parser has to model a position change with no cash flow — which breaks any ingestion pipeline that assumes every transaction has an amount.

**Ambiguous identifiers.** Sub-account numbers in the German fund system can also be valid securities identifiers (WKNs). The parser has to distinguish between "this is where the fund lives" and "this is which fund it is" from surrounding context.

These aren't hard computer science problems. They're hard *domain* problems — and they're invisible if you only look at the summary page.

## What the Tool Computes

Once the parser can reconstruct a portfolio from statements, the tool scores it. The health score starts at 100 and applies penalties across ten sections — each measuring a different dimension of portfolio risk:

**Concentration.** Not at the fund level — at the *underlying company* level. The tool looks through each ETF or fund to its constituents, weights them by portfolio allocation, and flags when the top three companies across all holdings exceed a threshold. A portfolio with one broad-market ETF holding 500 companies scores well here. A portfolio with five ETFs that all hold the same top-ten companies doesn't.

**Fee drag.** Weighted average TER across all positions, benchmarked against low-cost passive indexes. The tool computes the annual fee in absolute terms and the potential savings from switching to lower-cost alternatives. High-TER active funds are the first candidates for the exit plan.

**Complexity.** The number of distinct positions in the portfolio. The tool treats simplification as a goal — fewer positions means easier rebalancing, lower transaction costs, and less cognitive overhead. Above a configurable threshold, the score drops.

**Overlap.** What percentage of underlying companies appear in more than one fund? Redundant holdings mean you're paying multiple expense ratios for the same exposure.

**Geographic bias.** The split between US and non-US equity exposure, measured against a configurable baseline. Drift beyond a threshold costs points.

**Strategy and fund whitelisting.** Funds can be whitelisted — marked as core holdings to retain. The tool tracks allocation targets across categories and detects drift. Everything not on the whitelist becomes a candidate for the exit plan, prioritized by fee drag.

**Tax-optimized exit planning.** Germany's Freistellungsauftrag gives individuals a tax-free allowance on capital gains each year. The exit plan staggers sales across years to maximize that allowance — splitting non-whitelisted funds into immediate sales, next-year sales (to use next year's allowance), and notice-period sales. The tool tracks how much allowance remains and how much tax the staggering saves.

**Monte Carlo forecasting.** Percentile bands across 5, 10, and 20-year horizons, using historical volatility from the actual fund universe. The output isn't a prediction — it's a distribution. The shape of the distribution tells you things a single number never could.

The scoring philosophy is grounded in passive indexing, broad diversification, and low fees — but also in a deliberate constraint I gave it: the portfolio should be manageable by someone without a deep financial background. That constraint is why complexity is penalized. The tool wasn't just optimizing for returns — it was optimizing for comprehensibility.

The intellectual lineage isn't obscure. The fee drag penalty follows Bogle's argument that [costs are the most reliable predictor of fund performance](https://www.wiley.com/en-us/The+Little+Book+of+Common+Sense+Investing%3A+The+Only+Way+to+Guarantee+Your+Fair+Share+of+Stock+Market+Returns-p-9781119404507) — backed by the [SPIVA Scorecards](https://www.spglobal.com/spdji/en/research-insights/spiva/), which consistently show most active funds underperforming their benchmarks after fees. The diversification scoring draws on Markowitz's [portfolio selection](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1540-6261.1952.tb01525.x) framework. The simplification constraint echoes Malkiel's case in *A Random Walk Down Wall Street* for simple index portfolios — and [DeMiguel, Garlappi & Uppal's finding](https://doi.org/10.1093/rfs/hhm075) that naive equal-weight strategies often outperform complex optimized ones. The commodity hedge allocation takes a cue from Dalio's [All Weather](https://www.bridgewater.com/research-and-insights/the-all-weather-story) concept. None of this is novel. The tool just applies it mechanically.

## What the Tool Found

The portfolio I started with had over thirty individual positions — a mix of actively managed funds, ETFs, bonds, and other products accumulated over years of professional advice. The tool scored it poorly. High overlap between funds holding the same underlying companies. Elevated fee drag from active management. Complexity well above the threshold for practical self-management.

The tool's recommendation was drastic simplification: reduce to fewer than five positions anchored around a low-cost passive index core, with a small satellite allocation for commodity hedges as an inflation buffer. The exit plan staggered the liquidation of non-whitelisted funds across tax years to maximize the annual capital gains allowance.

The projected health score after executing the plan was substantially higher — not because the tool found some clever insight, but because it applied well-established principles mechanically. Broad diversification through a single low-cost index fund. Minimal overlap. Low fees. Geographic balance. These aren't secrets. They're the consensus view. The tool just made the gap between where the portfolio was and where the literature says it should be impossible to ignore.

## How It Was Built

Most of the implementation was done by AI coding agents. I described the domain, the edge cases, and the behavioral expectations — the agents wrote the code. The previous post on this blog, [*I Watched an AI Agent Refactor My Codebase While I Did Nothing*](/posts/2026-03-19-I_Watched_an_AI_Agent_Refactor_My_Codebase_While_I_Did_Nothing.html), documents one session in detail: an agent refactored the entire architecture, built a computation graph, wrote property-based contract tests, then ported the whole application from Go to Python — and deleted the Go code when it was done.

That session was representative, not exceptional. The tool started as a Go project. Agents built it, tested it, and eventually rewrote it in Python — all driven by behavioral scenarios in YAML that define what the tool does, not how. My role was defining those scenarios, validating the output against real data, and catching the things the agents missed. The health scoring algorithm, for example, only converged after I compared the two implementations side by side and noticed they disagreed.

The domain modeling — understanding German tax law, PDF structure, transaction semantics — was still mine to do. Agents are fast at writing code. They're not fast at reading Fondsabrechnungen and figuring out that a Vorabpauschale is actually two events.

## The Insight

German tax law — Vorabpauschale, Freistellungsauftrag, Verlustverrechnungstopf — isn't something I chose to implement. It's something the tax code forced me to implement. Every edge case in the PDF parser exists because the financial system has that edge case. The domain modeling was the real work. Once that was right, the rest was query logic — SQLite, a web UI, automatic price fetching, all running in Docker locally.

The answer to the title's question: yes, you can defer financial planning to a computational agent — if you're willing to do the domain work yourself first. The tool didn't invent a strategy. It applied established principles to real data and showed me what following the literature actually looks like for my specific portfolio. The gap between where I was and where I could be was wider than I expected.

---

*The gap was wider than I expected.*

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
