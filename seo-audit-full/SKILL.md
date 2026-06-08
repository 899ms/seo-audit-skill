---
name: seo-audit-full
description: >-
  Advanced full SEO audit skill. Runs its own full workflow from local scripts,
  including the basic SEO checks plus PageSpeed, social metadata, and advanced
  review modules when available.
metadata:
  author: Jeff
  version: "2.1"
---

# seo-audit-full — Advanced Full SEO Audit

This skill runs a full single-page SEO audit from the `seo-audit-full` directory.
It does not route to `seo-audit` when only a URL is provided.

---

## When to Use This Skill

Use `seo-audit-full` when the user asks for:

- "seo-audit-full"
- "full SEO audit"
- "advanced SEO audit"
- "technical SEO audit"
- "deep audit"
- "comprehensive SEO review"
- "audit everything"

URL-only requests are valid. When external datasets are not supplied, run the
full public-signal workflow and clearly note missing data sources in the report.

---

## Input Expected

| Input | Required | Notes |
|-------|----------|-------|
| Page URL | Yes | The primary page to audit |
| Primary keyword | Recommended | Improves content relevance scoring |
| PageSpeed API key | Yes | Ask the user for it at the start; run without it if unavailable |
| Raw HTML or page content | Optional | Enables more accurate content checks when supplied |
| GSC / crawl / analytics data | Optional | Include when supplied, otherwise mark unavailable |
| Competitor benchmark data | Optional | Include when supplied |

At the start of a full audit, ask the user for a PageSpeed Insights API key:

```
For PageSpeed checks, please provide a Google PageSpeed Insights API key.
Get one here: https://developers.google.com/speed/docs/insights/v5/get-started
Open "Acquiring and using an API key" → "Get a Key".
If you do not provide one, I will still run the audit and mark the PageSpeed
module with instructions for getting a key if the API is quota-limited.
```

---

## Architecture: Full = Core + Performance + Advanced

```
┌─────────────────────────────────────────────────────────────┐
│  seo-audit-full Workflow                                    │
│                                                             │
│  Phase 1: Run core scripts (./scripts/)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  check-site.py      → robots.txt, sitemap, 404, URL  │   │
│  │  check-page.py      → title, H1, meta desc, slug     │   │
│  │  check-schema.py    → JSON-LD validation              │   │
│  │  fetch-page.py      → raw HTML for analysis           │   │
│  └──────────────────────────────────────────────────────┘   │
│                          ↓                                  │
│  Phase 2: Run performance + full-only scripts (./scripts/)  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  check-pagespeed.py → Lighthouse scores + metrics     │   │
│  │  check-social.py    → OG Tags + Twitter Card          │   │
│  │  (more scripts added here as modules grow)            │   │
│  └──────────────────────────────────────────────────────┘   │
│                          ↓                                  │
│  Phase 3: LLM-only advanced checks                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  E-E-A-T content quality scoring                      │   │
│  │  Duplicate content signals                            │   │
│  │  Anchor text quality assessment                       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Output

Produce an **Advanced Full SEO Audit Report** by filling the template at
[assets/report-template.html](assets/report-template.html),
then **save it to a file — never print raw HTML to the terminal**.

**File naming:** `reports/<hostname>-<slug>-full-audit.html`
```
https://example.com/blog/best-tools → reports/example-com-blog-best-tools-full-audit.html
https://example.com/                → reports/example-com-full-audit.html
```

**After saving, tell the user:**
```
✅ Full Report saved → reports/example-com-full-audit.html
   Open it now? (yes / no)
```
If yes → run: `open reports/example-com-full-audit.html`

**Template placeholders** — fill each independently:

| Placeholder | Content |
|---|---|
| `{{summary_verdict}}` | One sentence: total checks run, how many failed/warned/passed |
| `{{summary_critical_html}}` | `<li>` per critical item, or `<li class="summary-empty">None</li>` |
| `{{summary_warnings_html}}` | `<li>` per warning item, or `<li class="summary-empty">None</li>` |
| `{{summary_passing_html}}` | `<li>` per passing check, or `<li class="summary-empty">None</li>` |
| `{{pagespeed_checks_html}}` | Full PageSpeed module using `check-pagespeed.py` output |
| `{{site_checks_html}}` | Site-level check tables |
| `{{eeat_checks_html}}` | E-E-A-T trust page table |
| `{{page_checks_html}}` | Page-level check tables, including full-only additions |
| `{{priority_actions_html}}` | Ordered priority action list |
| `{{insights_html}}` | Optional finding walkthrough cards |

---

## Scripts

Run full scripts from this directory. All output is structured JSON — use it
directly as evidence.

**Dependencies:** `pip install requests`

### Phase 1: Core scripts

```bash
# 1. site-level checks (robots.txt + sitemap.xml + 404 + URL canonicalization)
python scripts/check-site.py https://example.com

# 2. page-level checks (H1, title, meta description, canonical, URL slug)
python scripts/check-page.py https://example.com --keyword "primary keyword"

# 3. fetch raw HTML for downstream scripts
python scripts/fetch-page.py https://example.com --output /tmp/page.html

# 4. JSON-LD schema validation
python scripts/check-schema.py --file /tmp/page.html
```

### Phase 2: Full-only scripts

```bash
# 5. PageSpeed / Lighthouse checks
python scripts/check-pagespeed.py https://example.com --strategy mobile --timeout 180 --api-key "USER_PROVIDED_KEY"
# If no key was provided, run without --api-key and report any quota/API-key error.

# 6. Social tags: OG + Twitter Card validation
python scripts/check-social.py --file /tmp/page.html
# Or directly from URL:
python scripts/check-social.py https://example.com
```

Each script exits with code `0` (all pass/warn) or `1` (any fail/error).

PageSpeed can take 200 seconds. Use a 180-second timeout by default. If it
still times out, mark the Page Speed module as `error` and state that the
PageSpeed API timed out; do not treat that as confirmed page performance failure.
If PageSpeed fails because no API key was provided or Google returns a quota/API
key error, keep the audit running and render the PageSpeed module as `error` with
this instruction:
`Get a PageSpeed API key at https://developers.google.com/speed/docs/insights/v5/get-started → "Acquiring and using an API key" → "Get a Key".`

---

## Scope — Full Audit Check Whitelist

Full runs its own core checks plus the full-only items marked ★ below.

### Site-Level Checks (in `{{site_checks_html}}`)

Core checks:
- robots.txt · sitemap.xml · 404 Handling · URL Canonicalization · i18n / hreflang

### Page Speed Checks (in `{{pagespeed_checks_html}}`)

Full-only:
- Lighthouse category scores: Performance · Accessibility · Best Practices · SEO
- Lab metrics: FCP · LCP · TBT · CLS · Speed Index
- Final URL and screenshot availability

### E-E-A-T Checks (in `{{eeat_checks_html}}`)

Core checks:
- About Us · Contact · Privacy Policy · Terms of Service · Media/Partners (only if present)

### Page-Level Checks (in `{{page_checks_html}}`), output in this exact order:

Core checks:
URL Slug · Title Tag · Meta Description · H1 Tag · Canonical Tag · Image Alt Text ·
Word Count · Keyword Placement · Heading Structure · Internal Links · Schema (JSON-LD)

★ Full-only additions:
- **OG Tags** — og:title, og:description, og:image, og:type, og:url presence and validity
- **Twitter Card** — twitter:card type, title/description/image (with OG fallback detection)

---

## How to Use Script JSON Output

Same rules across full audit modules — map each field's `status` directly to the report check table:
- `status` → `pass` / `warn` / `fail` / `error` → badge in report
- `detail` → starting point for Evidence line
- Do not contradict script output unless you have additional observable evidence

**For `check-social.py` output:**
- `og.status` → OG Tags row status
- `twitter_card.status` → Twitter Card row status
- `og.fields.*` → individual field details for the detail cell
- `twitter_card.fields.*` → individual field details, note fallback fields

**For `check-pagespeed.py` output:**
- `status` → overall Page Speed status
- `category_status` → Lighthouse Scores badge status
- `metric_status` → Core Lab Metrics badge status
- `final_url` → Final URL line
- `categories.performance.score` → Performance score
- `categories.accessibility.score` → Accessibility score
- `categories["best-practices"].score` → Best Practices score
- `categories.seo.score` → SEO score
- `compact_metrics.fcp.display_value` → FCP
- `compact_metrics.lcp.display_value` → LCP
- `compact_metrics.tbt.display_value` → TBT
- `compact_metrics.cls.display_value` → CLS
- `compact_metrics.si.display_value` → Speed Index
- `screenshot` truthy → Screenshot available

Do not use overall `status` for the Lighthouse Scores badge. Category scores use
Lighthouse thresholds: 90–100 pass, 50–89 warn, 0–49 fail. Example: Performance
60, Accessibility 84, Best Practices 100, SEO 100 means `category_status` is
`warn`. If LCP or Speed Index fails, `metric_status` and overall `status` may be
`fail` while the Lighthouse Scores panel remains `warn`.

Inside `{{pagespeed_checks_html}}`, include a short `Priority Actions` list after
the score/metric cards. Keep it to 2–4 concise items based on failing or warning
PageSpeed fields:
- LCP slow → optimize hero media, preload critical image, reduce render-blocking CSS
- Speed Index slow → defer non-critical scripts and reduce above-the-fold JS/CSS
- TBT high → split long tasks and delay third-party tags
- Performance score warning/fail → prioritize the biggest Lighthouse opportunities

Always include the official diagnostic link:
`https://pagespeed.web.dev/`

---

## LLM Review Instructions

### Core LLM reviews

Resolve every `llm_review_required: true` field before writing the report:
H1 semantic judgment, Title keyword position, URL Slug evaluation, and Meta
Description quality must all receive an explicit judgment.

### Full-only LLM checks

**OG Tags quality (always review):**
```
og:title   : Does it differ meaningfully from <title>? It should be optimized for social sharing.
og:description : Is it compelling for social feeds? Different focus than meta description is OK.
og:image   : Is the URL an actual image path (not a page URL)?
```

**Twitter Card completeness:**
```
If twitter:card is "summary_large_image", twitter:image (or og:image fallback) must be
at least 300x157px. Flag if the image URL looks like a small icon or favicon.
```

---

## Recommended Workflow

1. **Acknowledge full scope** — confirm this is a full audit
2. **Infer primary keyword** — read the page H1, title, and first paragraph unless the user provided one
3. **Phase 1: Run core scripts** — check-site → check-page → fetch-page → check-schema
4. **Phase 2: Run full-only scripts** — check-pagespeed → check-social
5. **Core checks** — 404 handling, URL canonicalization, E-E-A-T trust pages, i18n/hreflang
6. **PageSpeed checks** — summarize Lighthouse category scores and lab metrics
7. **LLM-only advanced checks** — E-E-A-T content quality, duplicate content signals, anchor text quality
8. **Summarize findings** — Evidence / Impact / Fix format
9. **Priority actions** — top 5 highest-impact fixes with effort/impact tags
10. **Render report** — save to `reports/<hostname>-<slug>-full-audit.html`

---

## Report Detail Writing Rules

Use strict formatting:

**Pass → one short phrase. No lists, no elaboration.**

**Warn → one `<div class="detail-issue">` with ≤2 bullet points. One `<div class="detail-fix">`.**

**Fail → same as Warn. Lead with the exact failure.**

---

## Mandatory Finding Format

```
**Finding: [Finding Title]**

- **Evidence:** [Observable fact, data point, or marked assumption]
- **Impact:** [SEO / UX consequence]
- **Fix:** [Actionable recommendation with example]
```

For Priority Actions, add effort/impact tags:
```
1. [High Impact / Low Effort] Fix og:image — social shares currently show no preview.
```

---

## Reference Files

- Detailed audit modules and field definitions: [references/REFERENCE.md](references/REFERENCE.md)
- Final HTML report template: [assets/report-template.html](assets/report-template.html)
- PageSpeed validation script: [scripts/check-pagespeed.py](scripts/check-pagespeed.py)
- Social tags validation script: [scripts/check-social.py](scripts/check-social.py)
- Core scripts: [scripts/](scripts/) (check-site, check-page, check-schema, fetch-page)
