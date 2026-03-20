---
title: I Replaced My Financial Advisor With a Python Script
date: 2026-04-15
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: What I learned about retail financial advice by building the replacement — parsing German fund statements, modeling a tax system, and discovering that the hard part was never the code.
draft: true
---

My financial consultant sent me a letter. The custodian behind my portfolio was closing, and everything needed to move to a new broker. The consultant would handle the transfer. What they wouldn't handle — what they'd never handled — was answering the questions I actually cared about: *What am I paying in fees? Which funds are dragging? What's my real return after tax?*

I had years of quarterly fund statements sitting in a folder. Each one a record of what happened to my money. None of them connected to each other in any useful way.

So I built a tool that connects them.

## The Real Problem Is the PDF

German fund statements (Fondsabrechnungen) are multi-page PDFs with a specific structure: header block, one or more transaction blocks, tax summary footer. The transactions come in seven types — buys, sells, reinvestments, fee debits, custody fees, advance lump-sum tax sales, and gratuitous transfers — each with its own layout.

The common cases are straightforward. A buy is a date, a fund, an amount, a price, and a number of units. The interesting engineering problems are the edge cases:

**Compound tax transactions.** Germany's Vorabpauschale is a tax on unrealized gains, settled by a forced micro-sale of fund units. It's a single PDF block that represents two events: a tax liability and a position reduction. If the parser treats it as one thing, your holdings drift.

**Zero-amount transfers.** When units move between sub-accounts, no money changes hands. The parser has to model a position change with no cash flow — which breaks any ingestion pipeline that assumes every transaction has an amount.

**Ambiguous identifiers.** Sub-account numbers in the German fund system can also be valid securities identifiers (WKNs). The parser has to distinguish between "this is where the fund lives" and "this is which fund it is" from surrounding context.

These aren't hard computer science problems. They're hard *domain* problems — the kind that a financial advisor waves away because they only look at the summary, not the transactions.

## What an Advisor Actually Does

Here's what I learned by building the replacement: a retail financial advisor, for a straightforward portfolio, runs a spreadsheet. They track what you own, what you paid, and what it's worth. They might check allocation targets once a year. They charge a percentage for this.

The computational version of that job is more interesting:

**Health scoring.** A composite score from penalty sections — concentration risk, fee drag via TER analysis, geographic bias through look-through analysis of underlying holdings, category diversification. The score is a function of the portfolio state, not a judgment call. It changes when the data changes.

**Monte Carlo forecasting.** Percentile bands across multiple time horizons, using historical volatility from the actual fund universe rather than generic assumptions. The output isn't a prediction — it's a distribution. The shape of the distribution tells you things a single number never could.

**Tax-optimized exit planning.** Germany's Freistellungsauftrag gives individuals a tax-free allowance on capital gains each year. An optimal exit plan staggers sales across years to maximize that allowance. This is a constraint satisfaction problem, not a judgment call. A script solves it exactly; an advisor may not even mention it.

## The Insight

The hard part of replacing a financial advisor was never the code. It was understanding the domain well enough to model it. German tax law — Vorabpauschale, Freistellungsauftrag, Verlustverrechnungstopf — isn't something I chose to implement. It's something the tax code forced me to implement. Every edge case in the PDF parser exists because the financial system has that edge case.

Once the domain model was right, the rest was query logic. SQLite. A web UI. Automatic price fetching. The whole thing runs from a single Python file on a Raspberry Pi.

The lesson isn't "fire your financial advisor." It's that the value a retail advisor provides for a simple portfolio is almost entirely computational — and computation is cheap. The expensive part is knowing what to compute. For that, I had to read the PDFs myself.

---

*The complexity was in the domain, not the infrastructure.*
