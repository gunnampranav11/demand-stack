# Website Conversion Funnel Analysis + Landing Page Content Analysis

## Goal
Every week, pull website traffic and conversion data from GA4 to map the full visitor-to-conversion funnel. Identify which landing pages convert well and which don't. Cross-reference with Google Ads data to detect mismatches between ad copy promises and landing page content — a key driver of high bounce rates on paid search. Surface specific pages that need attention, not general observations.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. GA4 Data API**
- Property ID: `[YOUR_GA4_PROPERTY_ID]`
- Active conversion events: replace with your actual GA4 conversion event names
  <!-- # Replace with your actual GA4 conversion event names -->
  <!-- Examples: demo_request_submitted, trial_started, contact_form_submitted, purchase -->
- Ignore inactive/deprecated events: list any events you no longer track

**2. Website Content Crawler (Apify)**
- Actor ID: `apify/website-content-crawler`
- Used for: Crawling landing page content for ad copy vs. landing page mismatch detection

**3. Google Ads keyword data**
- Source: `.tmp/google_ads_keywords.csv` (produced by `directives/keyword_intelligence.md` → `execution/google_ads_pull.py`)
- Used for: Cross-referencing which keywords and ads drive traffic to which landing pages

---

## Step 1: GA4 Traffic and Conversion Pull

**What to pull:**

**Landing page report:**
- Landing page path (e.g., `/solutions`, `/demo`, `/pricing`)
- Sessions
- Engaged sessions
- Engagement rate
- Average session duration
- Bounce rate
- New users vs. returning users
- Conversions broken out by each active event
- Total conversions (sum of all active events)
- Conversion rate (total conversions ÷ sessions)

**Source/medium breakdown per landing page:**
For each landing page, also pull traffic broken out by source/medium:
- google / cpc (paid search)
- google / organic (organic search)
- linkedin / paid (LinkedIn ads — may appear as linkedin.com / referral or l.linkedin.com / referral)
- facebook / paid (Meta ads — may appear as facebook.com / referral or l.facebook.com / referral)
- direct / (none)
- referral sources
- email

This allows the analysis layer to compare conversion rates for the same landing page across different traffic sources.

**Filters:**
- Property ID: `[YOUR_GA4_PROPERTY_ID]`
- Date range: last 7 days
- All landing pages — pull everything, do not filter
- GA4 event name spacing: Use exact strings as configured in your GA4 property. Do not normalize.

**Save to:** `.tmp/ga4_conversion_funnel.csv`

---

## Step 2: Funnel Stage Mapping

**What to do:**
Map each landing page into a funnel stage based on the URL path:

| URL pattern | Funnel stage |
|---|---|
| `/` (homepage) | Top of Funnel — Awareness |
| `/blog/*` or `/resources/*` | Top of Funnel — Education |
| `/[solution-page-1]`, `/[solution-page-2]` | Mid Funnel — Solution |
| `/vs-*` or `/compare-*` or any comparison page | Mid Funnel — Evaluation |
| `/demo`, `/contact`, `/schedule`, `/pricing` | Bottom Funnel — Conversion |
| `/thank-you`, `/confirmation` | Post-Conversion |
| Everything else | Uncategorized |

**Note:** The URL patterns above are based on typical B2B SaaS website structures. Replace the solution page patterns (e.g., `/[solution-page-1]`) with your actual solution or product page paths. During initial testing, validate the actual URL paths on your domain and update this mapping in the directive if they don't match. The script should log any landing page that receives more than 50 sessions but doesn't match any pattern, so the mapping can be expanded.

**Save funnel stage as an additional column in:** `.tmp/ga4_conversion_funnel.csv`

---

## Step 3: High-Traffic Zero-Conversion Page Detection

**What to flag:**
- Any landing page with more than 50 sessions in the 7-day window and 0 total conversions across all active events
- Any landing page with more than 100 sessions and a conversion rate below 0.5%
- Any landing page where paid search (google / cpc) sessions are more than 20 but conversions are 0 — this directly indicates wasted Google Ads spend
- Any landing page with a bounce rate above 80% and more than 30 sessions

**For each flagged page, record:**
- Landing page path
- Sessions (total and by source)
- Bounce rate
- Conversion rate
- Estimated wasted spend (if paid search — cross-reference with `.tmp/google_ads_keywords.csv` to estimate how much was spent driving traffic to this page)
- Flag type: "zero-conversion", "low-conversion", "paid-zero-conversion", or "high-bounce"

**Save to:** `.tmp/flagged_landing_pages.csv`

---

## Step 4: Landing Page Content Crawl

**What to do:**
Crawl the actual content of flagged landing pages and top-traffic paid search landing pages to enable ad copy vs. landing page mismatch detection.

**Pages to crawl:**
1. All pages flagged in Step 3 (zero-conversion or low-conversion)
2. The top 10 landing pages by paid search (google / cpc) sessions, regardless of conversion rate
3. Deduplicate — if a page appears in both lists, crawl it once

This should result in approximately 10–20 pages per week.

**Run the Website Content Crawler (`apify/website-content-crawler`) on each page. For each crawled page, extract:**
- Full URL
- Page title
- H1 heading
- All H2 headings
- Full page text content
- Meta description
- CTA text (any buttons or links with action words like "Schedule", "Demo", "Contact", "Download", "Get Started")
- Whether a form exists on the page (yes/no)
- Whether a video exists on the page (yes/no)

**Save to:** `.tmp/landing_page_content.csv`

---

## Step 5: Ad Copy vs. Landing Page Cross-Reference

**What to do:**
Match Google Ads keywords and ad copy to the landing pages they drive traffic to, so the analysis layer can detect mismatches.

**Data needed:**
1. From `.tmp/google_ads_keywords.csv` (produced by `directives/keyword_intelligence.md`): keyword text, ad group name, campaign name, final URL (the landing page the ad sends traffic to)
2. From `.tmp/landing_page_content.csv` (produced in Step 4): landing page content, H1, H2s, CTA text

**Matching logic:**
- Match Google Ads keywords to landing pages by the final URL in the ad setup
- For each match, record: keyword text, ad group, campaign, landing page URL, landing page H1, landing page CTA text

**Note:** The actual mismatch detection (does the ad promise X but the landing page talks about Y?) is done by the analysis layer (`execution/analyze.py` using Claude API). The script just prepares the matched data.

**Save to:** `.tmp/ad_landing_page_matches.csv`

---

## Step 6: Week-Over-Week Comparison

**What to do:**
Compare this week's conversion funnel data against last week's snapshot (stored in `.tmp/conversion_funnel_snapshot.json`).

**Flag and report:**
- Landing pages with significant traffic drops (more than 30% decrease in sessions week over week)
- Landing pages with significant conversion rate changes (more than 50% increase or decrease)
- New landing pages receiving traffic that didn't exist last week (new pages published or new ad destinations)
- Landing pages that lost all traffic (possible page removed, redirect broken, or campaign paused)

**Save current state as:** `.tmp/conversion_funnel_snapshot.json`
**Save changes as:** `.tmp/conversion_funnel_changes.json`

On first run, save snapshot and skip change detection.

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/ga4_conversion_funnel.csv` | All landing pages with sessions, conversions, source/medium, and funnel stage |
| `.tmp/flagged_landing_pages.csv` | Pages with zero or low conversions, high bounce, or wasted paid spend |
| `.tmp/landing_page_content.csv` | Crawled content for flagged and top paid pages |
| `.tmp/ad_landing_page_matches.csv` | Google Ads keywords matched to their landing page content |
| `.tmp/conversion_funnel_snapshot.json` | Current funnel state for next week's comparison |
| `.tmp/conversion_funnel_changes.json` | Week-over-week changes in traffic and conversion rates |

---

## What the Analysis Layer Does With This Data

**For Section 1.7 and 2.7 of the weekly report (per vertical):**
- Full funnel visualization: sessions at each funnel stage (awareness → education → solution → evaluation → conversion), with conversion rates between stages
- Top converting landing pages: which pages drive the most conversions, and from which sources
- Worst performing landing pages: zero-conversion and high-bounce pages with specific session counts and wasted spend estimates
- Ad copy vs. landing page mismatch report: for each flagged mismatch, show the keyword/ad intent, what the landing page actually says, and a specific recommendation to align them
- Source comparison: for the same landing page, how does conversion rate differ between organic vs. paid vs. social traffic? This reveals whether the page content is the problem or the traffic quality is the problem.

**For Section 3 (Cross-Vertical Summary):**
- Total paid search spend going to zero-conversion pages (combines data from `directives/keyword_intelligence.md`)
- Top 3 landing pages most urgently needing improvement, ranked by wasted spend
- Week-over-week funnel health: is overall conversion rate improving or declining?
- New page alerts: pages receiving traffic for the first time this week

---

## Edge Cases and Notes

- **GA4 data delay:** GA4 data may be delayed by 24–48 hours. The most recent 1–2 days in the 7-day window may show incomplete data. This is normal and acceptable.
- **GA4 event name spacing:** Some GA4 event names may have unusual internal spacing or formatting — use the exact strings as configured in your GA4 property. Do not normalize.
- **Source/medium mapping:** LinkedIn and Meta traffic may appear under various source/medium combinations depending on UTM tagging. Common patterns: `linkedin.com / referral`, `l.linkedin.com / referral`, `facebook.com / referral`, `l.facebook.com / referral`. The script should group these by platform (LinkedIn, Meta, Google, Direct, Email, Other) as an additional column.
- **Landing page URL normalization:** GA4 reports landing page paths (e.g., `/demo`), not full URLs. The script should normalize by removing trailing slashes and query parameters so that `/demo`, `/demo/`, and `/demo?utm_source=google` are treated as the same page.
- **Apify costs for landing page crawl:** Approximately 10–20 pages per week at minimal cost (well under $0.10/week).
- **Website Content Crawler failures:** Some pages may block crawling. If a page fails to crawl, log it as "content unavailable" and continue. The analysis layer will note that ad-landing page mismatch analysis was not possible for that page.
- **Google Ads final URL:** Not all Google Ads keyword reports include the final URL. If the final URL is not available at the keyword level, pull ad-group-level or campaign-level final URLs as a fallback.
- **Dependency on keyword intelligence data:** Step 5 requires `.tmp/google_ads_keywords.csv` from `directives/keyword_intelligence.md`. The orchestrator must ensure that directive's scripts run before this module's scripts.
- **Funnel stage mapping validation:** The URL patterns in Step 2 are estimates. During initial testing, review your actual URL structure and update the mapping if needed. Log any high-traffic uncategorized pages so the mapping can be expanded.
- **First run:** No previous snapshot exists. Save current state, skip week-over-week comparison.

---

## Scripts This Directive Feeds

- `execution/ga4_pull.py` — Step 1
- `execution/conversion_funnel_pull.py` — Steps 2, 3, 5, and 6
- `execution/analyze.py` — Ad copy vs. landing page mismatch analysis (Step 4 content + Step 5 matching) and all reporting
