---
name: technical-seo-audit
description: Run a comprehensive, prioritized technical SEO audit using Ahrefs Site Audit MCP. Use this skill whenever the user asks for a technical SEO audit, site health check, crawl audit, indexability review, or wants to understand technical SEO issues for a client. Also trigger when the user mentions fixing 404s, redirects, canonicals, noindex, hreflang errors, page speed, schema validation, or site crawl problems — even if they don't use the phrase "technical SEO audit". Always use this skill for any Ahrefs-based site audit request.
---

# Technical SEO Audit Skill

Produces a prioritized technical SEO audit document from Ahrefs Site Audit data. Every finding is verified against raw source HTML before being reported. Output is a structured Markdown file saved to disk — ordered by business impact, not issue count.

---

## Step 0 — Setup

Ask the user (if not already known from context):
1. **Domain** — which site to audit (e.g. `azul.com`)
2. **Output path** — where to save the report (default: `./technical-seo-audit-[domain]-[date].md`)
3. **Any context?** — known priorities, pages under active development, regions or subsections to focus on or exclude

Then proceed. Do not wait for permission to start pulling data.

---

## Phase 1 — Project Discovery

Call `site-audit-projects` to find the Ahrefs project for the domain.

Extract and note:
- **Project ID** (needed for all subsequent calls)
- **Health score** (0–100)
- **Crawled URL count** — total, errors, warnings, notices
- **Last crawl date**

If no project exists for the domain, stop and tell the user they need to set up an Ahrefs Site Audit project for the domain first.

---

## Phase 2 — Issue Extraction

Call `site-audit-issues` with the project ID. This returns the full list of issue types with counts.

Filter for issues where `crawls_in_count > 0` (active issues in the current crawl). Ignore zero-count issues entirely.

Group by Ahrefs severity:
- **Error** — crawlability and indexability blockers
- **Warning** — on-page gaps and redirect problems
- **Notice** — lower-urgency improvements

Within each severity tier, sort by `crawls_in_count` descending. This gives you the volume landscape, but do not let volume override impact — a 404 with 67 external backlinks outranks a missing alt text on 2,000 pages. Use the prioritization framework in Phase 5 to reorder.

---

## Phase 3 — Page-Level Sampling

For each significant issue type, call `site-audit-page-explorer` to get the actual affected URLs with context.

### Critical: correct column names

These are the only valid column names. Using anything else returns a "column not found" error:

| Want | Use |
|---|---|
| Inbound internal links | `incoming_links` |
| Organic traffic estimate | `traffic` |
| Uncompressed page size | `size_uncompressed` |
| Compressed page size | `size` |
| HTTP redirect code | `redirect_code` |
| Noindex status | `page_is_noindex` |
| Canonical tag URL | `canonical_url` |
| Page title | `title` |
| H1 text | `h1` |
| Meta description | `meta_description` |
| Load time (ms) | `loading_time` |

### Standard query pattern

For every issue type, always include:
```
select: url, incoming_links, traffic
```

Then add issue-specific columns. Always order by `incoming_links:desc` first, then `traffic:desc`. This surfaces the highest-impact pages in every issue set — a 404 with 0 links and 0 traffic is trivially different from one with 67 external backlinks.

### Issue-specific columns to add

| Issue type | Add to select |
|---|---|
| 404 / 4XX errors | `redirect_code` |
| Redirects in sitemap | `redirect_code` |
| Noindex pages | `page_is_noindex`, `meta_description` |
| Canonical problems | `canonical_url` |
| Missing H1 | `h1` |
| Missing meta description | `meta_description` |
| Slow pages | `loading_time` |
| Large pages | `size_uncompressed` |
| Title too long | `title` |

### How many pages to pull per issue

Pull top 20–50 by `incoming_links:desc` for each major Error issue. For Warning/Notice issues, top 10–20 is sufficient unless there's a clear template pattern (e.g. all glossary pages missing H1 = one template fix). You are looking for the highest-impact instances and patterns, not an exhaustive URL list.

---

## Phase 4 — Verification (Do Not Skip)

**Ahrefs flags are hypotheses, not confirmed facts.** Before including any finding in the report, verify it from the source. This is the most important phase — the audit's credibility depends on it.

### Rule: what requires verification

| Claim type | Verification method |
|---|---|
| noindex tag present | Raw HTML extraction (see below) |
| Canonical URL value | Raw HTML extraction |
| H1 missing or present | Raw HTML extraction |
| Meta description value | Raw HTML extraction |
| Hreflang tag content | Raw HTML extraction |
| HTTP status (404, 301, 302) | WebFetch live check |
| Redirect destination URL | WebFetch live check |

### Raw HTML extraction method

Call `site-audit-page-content` with:
```
project_id: [id]
url: [url]
select: raw_html
```

Save the response to a temp file (e.g. `/tmp/page_head_check.txt`). The response is a JSON-encoded string with escaped characters. Double-unescape before parsing:

```python
import json, re

with open('/tmp/page_head_check.txt') as f:
    raw = f.read()

data = json.loads(raw)
html = data[0]['text'].replace('\\"', '"').replace('\\n', '\n').replace('\\t', '\t')

head_match = re.search(r'<head[^>]*>(.*?)</head>', html, re.DOTALL | re.IGNORECASE)
head = head_match.group(1) if head_match else ''

robots     = re.findall(r'<meta[^>]+robots[^>]+>', head, re.IGNORECASE)
canonical  = re.findall(r'<link[^>]+canonical[^>]+>', head, re.IGNORECASE)
title      = re.findall(r'<title[^>]*>(.*?)</title>', head, re.IGNORECASE | re.DOTALL)
meta_desc  = re.findall(r'<meta[^>]+name=["\']description["\'][^>]+>', head, re.IGNORECASE)
h1_tags    = re.findall(r'<h1[^>]*>(.*?)</h1>', html, re.IGNORECASE | re.DOTALL)
```

### Critical: Yoast/CMS robots directive pattern

CMS plugins like Yoast SEO embed multiple directives in a single comma-separated `content` attribute:

```html
<meta name='robots' content='max-image-preview:large, noindex,nofollow' />
```

`noindex` is present here — but it will not appear as a standalone value. Do not dismiss a noindex claim just because `content='noindex'` isn't found as an exact match. Parse the full content string and check for `noindex` as a substring.

### Canonical loop verification

When Ahrefs flags "canonical points to redirect," always verify both directions:
1. What does Page A's canonical tag say? (raw HTML)
2. Does that canonical destination redirect — and where to? (WebFetch)
3. Does the redirect destination match Page A's URL?

If yes: this is a **canonical loop** (worse than canonical-to-redirect — Googlebot cannot resolve it). Canonical loop = P1. Canonical-to-redirect with no loop = P2.

### WebFetch rules

WebFetch converts HTML to markdown, stripping all `<head>` content. Only use it for:
- Confirming an HTTP status code (200, 301, 302, 404)
- Confirming a redirect destination URL
- Checking visible body content

Never use WebFetch to verify meta robots, canonical tags, hreflang, meta descriptions, or any other head-level metadata.

---

## Phase 5 — Prioritization Framework

Order every finding by **business impact**, not issue count. The question for each issue is: *what is actually being lost today — rankings, link equity, organic traffic, or crawl efficiency?*

### P1 — Critical: Fix Immediately

These issues are actively costing rankings or link equity right now.

- **Noindex on pages with high internal link count or organic traffic** — link equity dead end; traffic at risk
- **Orphan pages with organic traffic** — pages ranking on domain authority alone with no internal authority flowing to them; positions fragile
- **404 pages with external backlinks** — link equity currently pointing at nothing; 301 redirect recovers it immediately
- **Canonical loops** — Googlebot cannot resolve authority; internal links accumulate to no effect
- **Redirect URLs with very high internal link counts in sitemap** — global nav pointing to a redirect = sitewide link equity waste

### P2 — High: Fix Within 2 Weeks

Template-level on-page gaps and redirect errors that suppress ranking across entire sections.

- **H1 missing from a section template** — especially when high-traffic pages are affected
- **Meta description missing at scale** — Google auto-generates from body text; CTR typically worse
- **Indexable pages absent from all sitemaps** — especially localized pages with backlinks
- **302 redirects for permanent URL changes** — Google does not transfer full PageRank through temporary redirects
- **Hreflang errors** — broken alternate signals confuse international ranking consolidation

### P3 — Medium: Fix Within 1 Month

Real impact but not bleeding link equity or blocking indexing today.

- **Slow pages on high-traffic URLs** — Core Web Vitals LCP impact; 47% of site slow = infrastructure-level, not per-page
- **Schema.org validation errors** — lost rich result eligibility (FAQ accordions, Product ratings)
- **Title too long on trafficked pages** — Google rewrites titles when they significantly exceed 60 chars
- **HTTP internal links on HTTPS site** — unnecessary redirect hops on every internal click; mixed-content signals
- **479 redirect URLs in XML sitemaps** — Google may deprioritize sitemap submissions with large redirect counts
- **Open Graph / social meta incomplete** — social share previews missing or incorrect

### P4 — Low: Backlog

Address after P1–P3 are resolved.

- **Missing alt text (accessibility + image search)** — widespread but per-image fix; template fix for new content first
- **Internal links pointing to redirect URLs** — systemic link equity drag; address after fixing the redirects themselves
- **Multiple H1 tags** — usually a theme/template bug; low ranking impact but worth cleaning
- **Redirect chains** — extra hops; secondary to fixing broken redirects and updating internal links
- **Pages with excessive external links** — link equity dilution; review commercial pages first

---

## Phase 6 — Write the Audit

### Document structure

```markdown
# Technical SEO Audit — [domain]
**Date:** [date]
**Source:** Ahrefs Site Audit, Project ID [id] (crawl completed: [date])
**Overall health score:** [n]/100
**Crawled URLs:** [n] | Errors: [n] | Warnings: [n] | Notices: [n]

---

## Executive Summary

[2–3 sentences on the overall picture. Then 3 named clusters:]
1. **[Cluster 1 name]** — [brief description of what's broken and what's at stake]
2. **[Cluster 2 name]** — [brief description]
3. **[Cluster 3 name]** — [brief description]

---

## Priority 1 — Critical: Fix Immediately

[Individual issues, each as a ### sub-section]

---

## Priority 2 — High: Fix Within 2 Weeks

[Individual issues]

---

## Priority 3 — Medium: Fix Within 1 Month

[Individual issues]

---

## Priority 4 — Low: Backlog

[Individual issues]

---

## Priority Summary

| ID | Issue | Affected Pages | Key Impact | Effort | Sprint |
|---|---|---|---|---|---|

---

## Sprint Plan

**Sprint 1 (Week 1–2): Critical fixes**
- ...

**Sprint 2 (Weeks 3–4): Template and redirect fixes**
- ...

**Sprint 3 (Month 2): Performance and schema**
- ...

**Ongoing:**
- ...
```

### Per-issue section format

Each issue gets a `###` sub-section with:

1. **Ahrefs flag(s) + count** — the exact flag name(s) and number of affected pages
2. **Affected pages** — the specific URLs that matter most, shown in a table with columns relevant to the issue (`incoming_links`, `traffic`, `loading_time`, etc.)
3. **Context and business impact** — explain *why* this matters in terms of rankings, link equity, traffic, or crawl efficiency. Name the stakes. Do not just describe the issue — quantify what is being lost.
4. **Fix** — specific, actionable steps. Include CMS-level detail where possible (e.g. "one template change in the glossary CMS resolves all 51 cases"). Name the files, plugins, or config paths where you know them.
5. **Effort** — one of: Very low (<1 hour), Low (1–4 hours), Medium (1 day), High (multi-day/sprint). Be honest.

### Writing rules

- **Specific always beats general.** "The /newsroom/ has 2,280 incoming internal links feeding into a noindexed page" is useful. "Several pages have noindex issues" is not.
- **Link equity and traffic numbers make priority legible.** A 404 with 67 external backlinks is a P1. A 404 with zero links and zero traffic is a P4. Show these numbers.
- **Template-level findings deserve special attention.** If the same issue exists on 51 pages because of a single template gap, say so — the effort is "fix one template" not "fix 51 pages."
- **Effort and impact together justify priority.** Spell out the ratio: "30 minutes to remove one tag, recovers 2,280 internal links and organic traffic."
- **Do not report unverified claims.** If you could not verify a finding from raw HTML or a live check, either verify it or omit it.

---

## Output

Save the completed audit to the agreed output path. Print a brief summary in chat: health score, top 3 P1 issues by business impact, and how many total issues were documented.

---

## Common Errors to Avoid

These are real errors from past audits — commit them to memory:

| Wrong | Right | Why |
|---|---|---|
| `inlinks_count` | `incoming_links` | Ahrefs column name — wrong name returns "column not found" |
| `organic_traffic` | `traffic` | Same — wrong column name |
| `size` for page weight | `size_uncompressed` | `size` is compressed; Googlebot's 2MB limit applies to uncompressed |
| WebFetch for noindex check | `site-audit-page-content` + Python | WebFetch strips `<head>` entirely |
| "noindex not found" | Check full `content=` attribute | Yoast embeds noindex in comma-separated list with other directives |
| "canonical points to redirect" → P2 | Verify both directions first | If it loops back, it's a canonical loop → P1 |
| Report issue count as severity | Check `incoming_links` and `traffic` | 1 URL with 67 external backlinks > 500 URLs with 0 |
