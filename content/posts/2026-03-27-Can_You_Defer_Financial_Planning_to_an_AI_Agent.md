---
title: Can You Defer Financial Planning to an AI Agent?
date: 2026-03-27
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: I built a tool to analyze fund statements and compare its strategy to professional advice. The hard part was modeling the domain, not writing the code.
draft: true
---

Keeping track of a diversified portfolio is hard to do in your head. Dozens of positions across ETFs, bonds, and other products — each with different fee structures, tax treatment, and risk profiles. I had professional advice managing this, but I couldn't easily answer basic questions: *What am I actually paying in fees? Which positions are underperforming? Is there a simpler approach that historical data and established financial literature would support?*

I wanted to find out. So I built a tool that ingests fund statements, models the domain, and runs the analysis — then compared its output to what professional advice had produced. The question wasn't whether to fire the advisor. It was whether a computational approach, grounded in industry literature and historical data, would arrive at similar conclusions or suggest something simpler.

## The Real Problem Is the PDF

German fund statements (Fondsabrechnungen) are multi-page PDFs with a specific structure: header block, one or more transaction blocks, tax summary footer. The transactions come in seven types — buys, sells, reinvestments, fee debits, custody fees, advance lump-sum tax sales, and gratuitous transfers — each with its own layout.

The common cases are straightforward. A buy is a date, a fund, an amount, a price, and a number of units. The interesting engineering problems are the edge cases:

**Compound tax transactions.** Germany's Vorabpauschale is a tax on unrealized gains, settled by a forced micro-sale of fund units. It's a single PDF block that represents two events: a tax liability and a position reduction. If the parser treats it as one thing, your holdings drift.

**Zero-amount transfers.** When units move between sub-accounts, no money changes hands. The parser has to model a position change with no cash flow — which breaks any ingestion pipeline that assumes every transaction has an amount.

**Ambiguous identifiers.** Sub-account numbers in the German fund system can also be valid securities identifiers (WKNs). The parser has to distinguish between "this is where the fund lives" and "this is which fund it is" from surrounding context.

These aren't hard computer science problems. They're hard *domain* problems — the kind that a financial advisor waves away because they only look at the summary, not the transactions.

## What an Advisor Actually Does

Here's what I learned by building the tool: a retail financial advisor, for a straightforward portfolio, runs a spreadsheet. They track what you own, what you paid, and what it's worth. They might check allocation targets once a year. They charge a percentage for this.

The computational version of that job is more interesting:

**Health scoring.** A composite score from penalty sections — concentration risk, fee drag via TER analysis, geographic bias through look-through analysis of underlying holdings, category diversification. The score is a function of the portfolio state, not a judgment call. It changes when the data changes.

**Monte Carlo forecasting.** Percentile bands across multiple time horizons, using historical volatility from the actual fund universe rather than generic assumptions. The output isn't a prediction — it's a distribution. The shape of the distribution tells you things a single number never could.

**Tax-optimized exit planning.** Germany's Freistellungsauftrag gives individuals a tax-free allowance on capital gains each year. An optimal exit plan staggers sales across years to maximize that allowance. This is a constraint satisfaction problem, not a judgment call. A script solves it exactly; an advisor may not even mention it.

## The Insight

The hard part of building this tool was never the code. It was understanding the domain well enough to model it. German tax law — Vorabpauschale, Freistellungsauftrag, Verlustverrechnungstopf — isn't something I chose to implement. It's something the tax code forced me to implement. Every edge case in the PDF parser exists because the financial system has that edge case.

Once the domain model was right, the rest was query logic. SQLite. A web UI. Automatic price fetching. The whole thing runs from a single Python file on a Raspberry Pi.

The lesson isn't "fire your financial advisor." It's that much of what a retail advisor provides for a simple portfolio is computational — and computation is cheap. The expensive part is knowing what to compute. For that, I had to read the PDFs myself.

---

*The complexity was in the domain, not the infrastructure.*

<!--
@agent-context
pillar: building-personal-tools
concepts:
  - structured PDF parsing as a domain modeling problem
  - German tax system as constraint satisfaction (Vorabpauschale, Freistellungsauftrag)
  - compound transactions (single document, multiple events)
  - retail financial advice as a computational problem
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
