# Technical SEO Audit

A Claude Code skill that produces a prioritized technical SEO audit from Ahrefs Site Audit data. Every finding is verified against raw source HTML before being reported. Output is a structured Markdown file saved to disk — ordered by business impact, not issue count.

## Requirements

- Claude Code with the [Ahrefs MCP](https://ahrefs.com) connected
- An Ahrefs Site Audit project already set up for the target domain

## Usage

```
/seo-content-audit <domain>
```

Or just ask Claude to run a technical SEO audit, site health check, crawl audit, or indexability review. The skill also triggers on mentions of fixing 404s, redirects, canonicals, noindex, hreflang errors, page speed, schema validation, or site crawl problems.

## What It Does

Works through six phases:

| Phase | What happens |
|---|---|
| **1 — Project Discovery** | Finds the Ahrefs Site Audit project, extracts health score and crawl summary |
| **2 — Issue Extraction** | Pulls all active issues, groups by Error / Warning / Notice, sorts by count |
| **3 — Page-Level Sampling** | Fetches affected URLs for each issue type, ordered by `incoming_links` then `traffic` |
| **4 — Verification** | Confirms every finding from raw HTML or live WebFetch before reporting |
| **5 — Prioritization** | Reorders by business impact using the P1–P4 framework |
| **6 — Audit Report** | Writes a structured Markdown report with sprint plan, saved to disk |

## Priority Framework

Issues are assigned P1–P4 based on what is actually being lost — rankings, link equity, organic traffic, or crawl efficiency:

- **P1 — Critical (fix immediately):** Noindex on high-traffic pages, 404s with external backlinks, canonical loops, orphan pages with organic traffic
- **P2 — High (fix within 2 weeks):** Missing H1/meta description at template scale, 302s for permanent redirects, hreflang errors, pages missing from sitemaps
- **P3 — Medium (fix within 1 month):** Slow pages, schema validation errors, title length issues, HTTP internal links on HTTPS sites
- **P4 — Low (backlog):** Missing alt text, internal links to redirects, multiple H1s, redirect chains

## Verification Rules

Ahrefs flags are treated as hypotheses, not confirmed facts. Before any finding appears in the report:

- **Noindex, canonical, H1, meta description, hreflang** — verified by fetching `raw_html` via `site-audit-page-content` and parsing the `<head>`
- **HTTP status and redirect destination** — verified via live WebFetch
- **Canonical loops** — both directions verified: what does the canonical tag say, and does that destination redirect back?

WebFetch is never used for head-level metadata — it strips `<head>` entirely.

## Output

A Markdown audit report containing:
- Executive summary with 3 named issue clusters
- Per-issue sections with affected URLs, business impact, fix instructions, and effort estimate
- Priority summary table
- Sprint plan (3 sprints + ongoing)

Saved to `./technical-seo-audit-[domain]-[date].md` by default.

## Evals

`evals/evals.json` contains test cases for validating skill output quality.
