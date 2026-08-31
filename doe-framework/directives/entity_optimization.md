# Entity Optimization Check

## Goal
Every two months, audit your company's web presence for entity-level SEO health: schema markup (structured data), Google Knowledge Graph presence, and brand consistency across the web. Also check key competitors for comparison. This module ensures your digital entity is properly defined and recognized by search engines, which affects rankings, rich snippets, and brand authority.

---

## When This Runs
- **Frequency:** Bi-monthly (every 8 weeks, as part of the main orchestrator)
- **Data window:** Current snapshot (point-in-time)
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. Website Content Crawler (Apify)**
- Actor ID: `apify/website-content-crawler`
- Used for: Crawling your company and competitor pages to extract schema markup, meta tags, and page content

**2. Google Search Scraper (Apify)**
- Actor ID: `apify/google-search-scraper`
- Used for: Checking Knowledge Graph presence by searching brand names and analyzing SERP features

---

## Step 1: Schema Markup Audit (Your Company)

**What to do:**
Run the Website Content Crawler (`apify/website-content-crawler`) on all key pages to extract structured data (schema markup / JSON-LD).

**Pages to crawl:**
<!-- Replace with your actual key pages -->
- `https://[YOUR_DOMAIN]` (homepage)
- `https://[YOUR_DOMAIN]/about`
- `https://[YOUR_DOMAIN]/[solution-page-1]`
- `https://[YOUR_DOMAIN]/[solution-page-2]`
- `https://[YOUR_DOMAIN]/[solution-page-3]`
- `https://[YOUR_DOMAIN]/pricing` (if applicable)

**For each page, extract:**
- Full page HTML (raw source needed to find JSON-LD blocks)
- All `<script type="application/ld+json">` blocks (this is where schema markup lives)
- Open Graph meta tags (`og:title`, `og:description`, `og:image`, `og:type`, `og:url`)
- Twitter Card meta tags (`twitter:card`, `twitter:title`, `twitter:description`)
- Canonical URL (`<link rel="canonical">`)
- Page title (`<title>` tag)
- Meta description (`<meta name="description">`)

**Schema types to check for:**
- `Organization` — company name, logo, URL, social profiles, founding date
- `LocalBusiness` — address, phone, hours, geo coordinates (relevant if GBP is meant to rank locally)
- `WebSite` — site name, URL, search action
- `Product` or `SoftwareApplication` — product descriptions, features, pricing
- `BreadcrumbList` — navigation structure
- `FAQPage` — FAQ sections
- `Article` or `BlogPosting` — blog content

**For each page, record:**
- Page URL
- Schema types found (list all)
- Schema types missing (compared against the recommended set above)
- Whether the schema is valid JSON-LD (basic syntax check — does it parse as valid JSON?)
- Key fields present or missing within each schema type (e.g., Organization schema exists but is missing `logo` or `sameAs` social links)
- Open Graph completeness (are all 4 core OG tags present?)
- Canonical URL present and correct

**Save to:** `.tmp/company_schema_audit.csv`

---

## Step 2: Schema Markup Audit (Competitors)

**What to do:**
Run the Website Content Crawler on competitor homepages to compare schema implementation.

**Competitor pages to crawl (replace with your actual competitors):**
- `https://[competitor1-domain.com]`
- `https://[competitor2-domain.com]`
- `https://[competitor3-domain.com]`
- `https://[competitor4-domain.com]`
- `https://[competitor5-domain.com]`

**For each page, extract the same data as Step 1:**
- All JSON-LD blocks
- Schema types found
- Open Graph tags
- Meta description

**Save to:** `.tmp/competitor_schema_audit.csv`

---

## Step 3: Knowledge Graph Check

**What to do:**
Run the Google Search Scraper (`apify/google-search-scraper`) for brand name searches to check if a Knowledge Graph panel appears in the SERP.

**Search list (your company + key competitors):**
- [YOUR_COMPANY]
- [YOUR_COMPANY] Inc (or LLC, or your legal name)
- [COMPETITOR_1]
- [COMPETITOR_2]
- [COMPETITOR_3]
- [COMPETITOR_4]
- [COMPETITOR_5]

**Settings:**
- Country: United States
- Language: English
- Results per page: 10
- Max pages: 1

**For each search, check:**
- Does a Knowledge Graph panel appear on the right side of the SERP?
- If yes, what information does it show? (Company name, description, logo, website, social profiles, employees, founded date)
- If no, record as "No Knowledge Graph panel found"

**Note:** The Google Search Scraper may not reliably extract Knowledge Graph data — it depends on the actor's capabilities. If Knowledge Graph data is not returned, the script should record "Knowledge Graph check inconclusive — manual verification recommended" for that search. Do not fail the module over this.

**Save to:** `.tmp/knowledge_graph_check.csv`

---

## Step 4: Brand Consistency Check

**What to do:**
Using the data already collected in Steps 1–3, plus data from other modules, check for brand consistency issues.

**What to check:**
- **Company name consistency:** Is your company name spelled consistently across the website, schema markup, GBP listing, G2, Capterra, and LinkedIn? Flag any variations (e.g., different capitalization, abbreviations, legal suffixes)
- **Description consistency:** Compare the company description in schema markup, GBP listing (from `directives/gbp_analysis.md` data if available), review profile pages, and LinkedIn page. Flag major inconsistencies.
- **Logo presence:** Is a logo specified in the Organization schema? Does the GBP listing have a logo? Is it the same logo?
- **Social profile links:** Are social media links included in the Organization schema `sameAs` field? Do they match the actual profiles (LinkedIn, Twitter/X, Facebook, YouTube)?
- **Address consistency:** Compare the address in schema markup (if `LocalBusiness` schema exists) against the GBP listing address. Flag mismatches.
- **NAP consistency (Name, Address, Phone):** Are name, address, and phone number consistent across all touchpoints?

**Data sources for this check:**
- Schema markup data from Steps 1–2
- GBP data from `.tmp/gbp_raw_data.csv` (from `directives/gbp_analysis.md` — may not exist if GBP hasn't run recently)
- LinkedIn data from `.tmp/company_linkedin_page_metrics.csv` (from `directives/linkedin_organic.md`)

If any of these source files don't exist, skip that comparison and note it. The check works with whatever data is available.

**Save to:** `.tmp/brand_consistency_check.csv`

---

## Step 5: Compare Against Previous Audit

**What to do:**
Compare this audit's results against the previous bi-monthly snapshot (stored in `.tmp/entity_snapshot.json`).

**Flag and report:**
- **New schema added:** Any schema type that your company added since the last audit
- **Schema removed:** Any schema type that was present last audit but is now missing
- **Competitor changes:** Any competitor that added or removed schema types
- **Knowledge Graph changes:** Any brand that gained or lost a Knowledge Graph panel
- **Brand consistency changes:** Any new inconsistency or any fixed inconsistency

**Save current state as:** `.tmp/entity_snapshot.json`
**Save changes as:** `.tmp/entity_changes.json`

On first run, save snapshot and skip change detection.

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/company_schema_audit.csv` | Schema markup audit for all your company pages |
| `.tmp/competitor_schema_audit.csv` | Schema markup audit for competitor homepages |
| `.tmp/knowledge_graph_check.csv` | Knowledge Graph presence check for brand searches |
| `.tmp/brand_consistency_check.csv` | Brand consistency across all touchpoints |
| `.tmp/entity_snapshot.json` | Current entity state for next audit's comparison |
| `.tmp/entity_changes.json` | Changes since last audit |

---

## What the Analysis Layer Does With This Data

**For Section 4 of the weekly report (bi-monthly only):**

**Schema Markup Status:**
- Which schema types your company currently has vs. which are missing
- Priority recommendation: which schema types to add first, based on what competitors have and you don't
- Specific implementation guidance: "Add Organization schema to homepage with these fields: name, url, logo, sameAs, description, foundingDate"
- Competitor comparison: table showing which schema types each competitor has

**Knowledge Graph Presence:**
- Whether your company has a Knowledge Graph panel
- If not, what steps are needed to get one (typically: consistent schema markup, Wikipedia page, Wikidata entry, strong branded search volume)
- Which competitors have Knowledge Graph panels

**Brand Consistency:**
- List of all inconsistencies found, ranked by impact
- Specific fix for each inconsistency
- NAP consistency score: how many touchpoints match vs. don't

**Changes Since Last Audit:**
- What improved since the last bi-monthly check
- What regressed or needs attention

---

## Edge Cases and Notes

- **Bi-monthly scheduling:** The orchestrator (`execution/main.py`) must track when this module last ran. Use a simple check: if the last entity check was more than 55 days ago, run. If not, skip. This gives a buffer around the 60-day (bi-monthly) target.
- **Apify costs:** Your key pages + competitor pages = page crawls + SERP searches. Minimal cost — well under $0.50 per run. Since this runs bi-monthly, the cost is negligible.
- **Website Content Crawler returns text, not raw HTML:** Some Apify website crawlers return cleaned text, not raw HTML. If raw HTML is not available, JSON-LD blocks may not be extractable by the crawler alone. In this case, the script should make direct HTTP requests (using Python's `requests` library) to fetch raw HTML for the JSON-LD extraction, and use the Apify crawler only for content analysis.
- **JSON-LD parsing:** Use Python's `json` module to parse JSON-LD blocks. If a block fails to parse, flag it as "invalid JSON-LD" and include the raw text in the output for manual review.
- **Knowledge Graph detection limitations:** The Google Search Scraper may not reliably differentiate Knowledge Graph panels from other SERP features. If the data is ambiguous, flag it for manual verification rather than making assumptions.
- **GBP data may not exist:** GBP analysis runs bi-weekly, and this module runs bi-monthly. The GBP data file may or may not exist depending on timing. If `.tmp/gbp_raw_data.csv` doesn't exist, skip the GBP portions of the brand consistency check and note it.
- **Competitor pages may block crawling:** If a competitor page blocks crawling, log it as "blocked" and skip. The schema comparison works with whatever pages were successfully crawled.
- **First run:** No previous snapshot exists. Save current state and skip change detection. All analysis produces full results on first run.

---

## Scripts This Directive Feeds

- `execution/entity_check.py` — Steps 1, 2, 3, 4, and 5
- `execution/analyze.py` — All analysis layer reporting
