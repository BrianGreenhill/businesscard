# Editorial Guidelines

This document defines the editorial direction for briangreenhill.net. It is the source of truth for any agent or human creating, editing, or reviewing content for this site.

## Mission

Write about how software gets built, tested, and operated — from the perspective of someone who does all three. The blog exists to share lessons from real projects. It is not a brand, a newsletter, or a sales funnel.

## Audience

Engineers and technical leaders who build and operate software. They want concrete details, honest assessments, and transferable ideas — not thought leadership or engagement bait.

## Content Pillars

Every post must fit clearly into one of these three themes:

1. **Reliability & operations** — SRE practices, observability, failure modes, monitoring
2. **Building personal tools** — side projects, the decisions behind them, what worked and what didn't
3. **AI-assisted development** — how coding agents change the development workflow, with evidence

If a post doesn't fit one of these pillars, it doesn't belong on this site.

## Voice

- **First person, practitioner tone.** The author did the thing and is reporting what happened.
- **Show, don't tell.** Evidence over opinion. Diffs over advice. Before/after over proclamation.
- **Precise but accessible.** Use technical terms when they're the right word. A senior engineer from a different stack should be able to follow.
- **Understated.** No hype, no superlatives, no "game-changing." Let the evidence speak.
- **Honest about limits.** Say what didn't work. Say what you don't know.

## Content Boundaries

### Never publish

- **Personal financial information.** No account numbers, balances, portfolio values, fund names, broker names, or anything that reveals financial position. Technical concepts (e.g., "tax-loss harvesting," "Monte Carlo simulation") are fine. Specific numbers are not.
- **Employer-confidential information.** No internal systems, incidents, metrics, or architecture unless already public.
- **Other people's information.** No names, handles, or identifying details without explicit consent.
- **Security vulnerabilities.** Nothing unpatched or undisclosed.
- **Named criticisms of specific companies or people.** Write about patterns and systems, not grievances.

### Always do

- **Generalize from the specific.** The project is context, not content. The reader should take away a technique, a pattern, or a question — not details about the author's personal life.
- **Use representative examples, not real data.** Code blocks use invented sample data. Never paste real transactions, logs with real hostnames, or output with real identifiers.
- **Privacy pass before publish.** Search every post for account numbers, names, URLs, hostnames, and anything that connects the writing to a specific person, company, or financial position.

## Article Structure

Every post follows this shape:

1. **Context** (1–2 paragraphs) — What situation led to this? What question was being answered?
2. **The work** (main body) — What happened, in chronological or logical order. Code blocks earn their place — each one makes a specific point.
3. **Reflection** (1–2 paragraphs) — What was learned? What's the transferable insight?
4. **Closing line** — One sentence. Concrete. No call to action.

## Front Matter

Every post uses this YAML front matter:

```yaml
---
title: Post Title Here
date: YYYY-MM-DD
author: Brian Greenhill
author_image: https://avatars.githubusercontent.com/u/1642339?v=4
description: A single sentence that works as both SEO meta description and social preview.
draft: true          # remove or set to false to publish
---
```

- `draft: true` excludes the post from site generation. New posts start as drafts.
- `description` should be a complete sentence under 160 characters.
- `date` is the intended publication date.

## Cadence

One post per month. Write when there's a real artifact — a project milestone, an experiment result, a production lesson. Missing a month is better than publishing something empty.

## Site Generation

The site is built with [ssg](https://github.com/BrianGreenhill/ssg), a custom static site generator.

```bash
ssg generate          # build the site to public/
ssg watch -p 9090     # dev server with live reload
ssg post              # scaffold a new draft post
```

Templates live in `themes/default/`. Asset paths use absolute `/assets/...` URLs. Posts with `draft: true` in front matter are excluded from generation.

## For Agents

If you are an AI agent creating or editing content for this site:

1. **Read this file first.** It defines what is and isn't acceptable.
2. **Follow the content boundaries strictly.** When in doubt, leave it out. Generalize.
3. **Run a privacy pass** on any content you generate. Search for proper nouns, numbers that could be real data, and anything personally identifying.
4. **Match the voice.** Read the existing posts in `content/posts/` before writing new ones. The tone is understated and evidence-driven.
5. **Use `draft: true`** for all new posts. A human publishes by setting `draft: false`.
6. **Run `ssg generate`** after any content change to verify the site builds cleanly.
7. **Don't invent facts.** If you need specific details about a project, ask or check the source repo. Don't fabricate commit counts, line counts, or timelines.
