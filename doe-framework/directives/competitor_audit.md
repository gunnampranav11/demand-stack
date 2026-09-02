# Competitor Keyword & Ad Audit + Content Gap Analysis

## Goal
Every week, scan SERP rankings for all your ICP keywords to see where competitors rank vs. your company. Scrape competitor ad activity on Meta and LinkedIn. Monitor competitor LinkedIn organic posts for messaging themes and engagement. Crawl the top-ranking pages for each ICP keyword to identify content gaps - what type of content ranks, what format it uses, and what you need to create to compete. All competitive intelligence is gathered via Apify actors, not direct APIs.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Current snapshot (SERP and ad data are point-in-time, not historical)
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources (All Apify Actors)

**1. Google Search Scraper**
- Actor ID: `apify/google-search-scraper`
- Used for: SERP rankings for all ICP keywords

**2. Facebook Ad Library Scraper**
- Actor ID: `apify/facebook-ads-scraper`
- Used for: Competitor ad activity on Meta/Facebook/Instagram

**3. LinkedIn Ads Scraper**
- Actor ID: `[YOUR_LINKEDIN_ADS_ACTOR_ID]` - search Apify Store for "LinkedIn ad library scraper" and use the current best-rated option. As of May 2026: `silva95gustavo/linkedin-ad-library-scraper`
- Used for: Competitor LinkedIn paid ad activity

**4. LinkedIn Posts Scraper**
- Actor ID: `[YOUR_LINKEDIN_POSTS_ACTOR_ID]` - search Apify Store for "LinkedIn company posts scraper" and use the current best-rated option. As of May 2026: `harvestapi/linkedin-company-posts`
- Used for: Competitor LinkedIn organic post content and engagement

**5. Website Content Crawler**
- Actor ID: `apify/website-content-crawler`
- Used for: Content gap analysis - crawling top-ranking pages to analyze content type, structure, and format

**6. Competitor lists**
- Source: `config/competitor_lists.py`
- Contains all competitor names and domains organized by sub-vertical

---

## Step 1: SERP Scan for All ICP Keywords

**What to do:**
Run the Google Search Scraper for all your ICP keywords (defined in `config/icp_taxonomy.py`). For each keyword, pull the top 20 organic results and any paid ads showing.

**Settings per keyword:**
- Country: United States (adjust for your target market)
- Language: English
- Results per page: 10
- Max pages: 2 (to capture positions 1-20)

**Your ICP keyword list:**
<!-- Replace with your actual ICP keywords, organized by vertical -->
<!-- Example structure: -->

# Replace with your actual ICP keywords, organized by your verticals from config/icp_taxonomy.py
Vertical 1 (e.g., [YOUR_VERTICAL_1]):
- [keyword 1]
- [keyword 2]
- [keyword 3]

Vertical 2 (e.g., [YOUR_VERTICAL_2]):
- [keyword 1]
- [keyword 2]

Generic / broad:
- [keyword 1]
- [keyword 2]

Competitor brand keywords (to track if you appear in competitor searches):
- [Competitor 1 name]
- [Competitor 2 name]

**For each keyword, record:**
- Keyword text
- Vertical category
- Organic results: rank position, URL, domain, page title, snippet text
- Whether your company appears in the results (and at what position)
- Whether any known competitor (from `config/competitor_lists.py`) appears (and at what position)
- Paid ads: advertiser name, ad headline, ad description, display URL, ad position

**Compare against previous week's snapshot (stored in `.tmp/serp_snapshot.json`):**
- Flag any competitor that entered the top 10 for a keyword where they were not present last week
- Flag any competitor that dropped out of the top 10
- Flag any keyword where your company's position changed by more than 3 positions (up or down)
- Flag any new advertiser running paid ads on ICP keywords
- On first run, save snapshot and skip change detection

**Save to:** `.tmp/serp_results.csv` and `.tmp/serp_snapshot.json`

---

## Step 2: Competitor Ad Library Scan (Meta)

**What to do:**
Run the Facebook Ad Library Scraper for each competitor plus your own company, plus generic ICP keyword searches.

**Search list:**
<!-- Replace with your actual competitors from config/competitor_lists.py -->

Company name searches (replace with your actual competitors):
- [COMPETITOR_1]
- [COMPETITOR_2]
- [COMPETITOR_3]
- [COMPETITOR_4]
- [COMPETITOR_5]
- [YOUR_COMPANY]

Generic keyword searches (replace with your own ICP keywords):
- [your product category keyword 1]
- [your product category keyword 2]
- [your vertical keyword 1]
- [your vertical keyword 2]
- [your vertical keyword 3]

**For each result, record:**
- Advertiser name
- Ad creative text (headline, body, CTA)
- Ad format (image, video, carousel)
- Platform (Facebook, Instagram, or both)
- Active status
- Date started (if available)
- Landing page URL

**Compare against previous week's snapshot (stored in `.tmp/meta_ad_snapshot.json`):**
- Flag any competitor running new ads that did not exist last week
- Flag any competitor that stopped running ads
- Flag new messaging themes or angles
- On first run, save snapshot and skip change detection

**Save to:** `.tmp/competitor_meta_ads.csv` and `.tmp/meta_ad_snapshot.json`

---

## Step 3: Competitor Ad Scan (LinkedIn)

**What to do:**
Run the LinkedIn Ads Scraper (`[YOUR_LINKEDIN_ADS_ACTOR_ID]`) for each competitor plus your own company.

**Search list (company names - replace with your actual competitors):**
- [COMPETITOR_1]
- [COMPETITOR_2]
- [COMPETITOR_3]
- [COMPETITOR_4]
- [COMPETITOR_5]
- [YOUR_COMPANY]

**For each result, record:**
- Advertiser name
- Ad creative text (headline, body, CTA)
- Ad format
- Landing page URL
- Engagement metrics (if available from the scraper)

**Compare against previous week's snapshot (stored in `.tmp/linkedin_ad_snapshot.json`):**
- Flag new competitor ads
- Flag competitors that stopped running LinkedIn ads
- Flag new messaging themes or angles
- On first run, save snapshot and skip change detection

**Save to:** `.tmp/competitor_linkedin_ads.csv` and `.tmp/linkedin_ad_snapshot.json`

---

## Step 4: Competitor LinkedIn Organic Posts Scan

**What to do:**
Run the LinkedIn Posts Scraper (`[YOUR_LINKEDIN_POSTS_ACTOR_ID]`) for each competitor plus your own company to track their organic content strategy.

**Company LinkedIn pages to scrape (replace with your actual competitors):**
- [YOUR_COMPANY]: `https://www.linkedin.com/company/[your-company-slug]/`
- [COMPETITOR_1]: search by company name
- [COMPETITOR_2]: search by company name
- [COMPETITOR_3]: search by company name
- [COMPETITOR_4]: search by company name
- [COMPETITOR_5]: search by company name

**For each company, pull the most recent posts from the last 7 days. For each post, record:**
- Company name
- Post date
- Post content (full text)
- Post type (text only, image, video, document/carousel, article, poll)
- Reactions count
- Comments count
- Shares count (if available)
- Post URL

**Compare against previous week's snapshot (stored in `.tmp/linkedin_posts_snapshot.json`):**
- Flag competitors that significantly increased posting frequency (e.g., went from 1 post/week to 4+)
- Flag any competitor post with unusually high engagement (more than 2x their average reactions)
- Flag new messaging themes or product announcements
- On first run, save snapshot and skip change detection

**Note:** Your own company's posts are scraped here for competitive comparison purposes. Detailed analysis of your own LinkedIn organic performance (engagement rates, follower growth, thought leader posts) is handled separately in `directives/linkedin_organic.md`.

**Save to:** `.tmp/competitor_linkedin_posts.csv` and `.tmp/linkedin_posts_snapshot.json`

---

## Step 5: Content Gap Analysis

**Why this matters:** For each ICP keyword, the pages that rank on page 1 tell you what Google considers the best content for that query. Understanding the content format, structure, and angle of top-ranking pages reveals what you need to create to compete.

**What to do:**
1. From the SERP results in Step 1, take the top 3 organic results for each non-brand ICP keyword (skip competitor brand keywords - content gap analysis does not apply to branded searches)
2. Run the Website Content Crawler (`apify/website-content-crawler`) on each of those top-ranking URLs
3. For each crawled page, extract:
   - URL
   - Page title
   - Raw page content (full text)
   - All heading tags (H1, H2, H3)
   - Meta description
   - Whether the page includes structured data / schema markup

**Note:** The actual content type classification (blog post vs. product page vs. comparison page), word count estimation, angle summarization, and content recommendations are done by the analysis layer (`execution/analyze.py` using Claude API). The crawler just pulls the raw page content.

**Save to:** `.tmp/content_gap_pages.csv`

---

## Step 6: Cross-Reference with Competitor Lists

Using `config/competitor_lists.py`, tag every SERP result, ad, and LinkedIn post with the competitor's sub-vertical:

**Tagging rules:**
- Match the result domain against known competitor domains in your competitor lists
- Tag each result with the competitor's vertical(s)
- If a domain appears in multiple sub-vertical lists, tag with ALL matching verticals (comma-separated)
- If the domain is your own company, tag as `[YOUR_COMPANY]`
- If the domain does not match any known competitor, tag as `Unknown / Non-competitor`

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/serp_results.csv` | Full SERP data for all keywords with competitor tags |
| `.tmp/serp_snapshot.json` | Current SERP state for next week's comparison |
| `.tmp/serp_changes.json` | Competitor ranking changes, new entrants, position shifts |
| `.tmp/competitor_meta_ads.csv` | All Meta ad activity by competitor |
| `.tmp/meta_ad_snapshot.json` | Current Meta ad state for next week's comparison |
| `.tmp/competitor_linkedin_ads.csv` | All LinkedIn ad activity by competitor |
| `.tmp/linkedin_ad_snapshot.json` | Current LinkedIn ad state for next week's comparison |
| `.tmp/competitor_linkedin_posts.csv` | All competitor LinkedIn organic posts with engagement metrics |
| `.tmp/linkedin_posts_snapshot.json` | Current LinkedIn posts state for next week's comparison |
| `.tmp/content_gap_pages.csv` | Raw page content from top 3 results per non-brand ICP keyword |

---

## What the Analysis Layer Does With This Data

**For Section 1.3 and 2.3 of the weekly report (per vertical):**
- Competitor SERP presence: which competitors rank for which ICP keywords, and at what positions
- Your company vs. competitor ranking comparison per keyword
- Competitor ranking changes from last week (who moved up, who moved down, who is new)
- Competitor ad messaging analysis: what themes, CTAs, and angles competitors are using on Meta and LinkedIn (paid)
- Competitor organic messaging analysis: what topics and themes competitors are pushing on LinkedIn (organic), which posts are getting high engagement
- New competitor ad alerts: who started or stopped advertising this week
- Content gap analysis per keyword: what content type ranks (blog vs. product page vs. comparison), what format works, and what you should create to compete

**For Section 3 (Cross-Vertical Summary):**
- Competitor movement alerts: any competitor making significant SERP gains across multiple keywords
- Ad spend signals: competitors running aggressive new ad campaigns
- Competitor messaging trends: emerging themes or shifts across LinkedIn organic + paid + Meta
- Content gap priorities: top 5 content pieces to create, ranked by keyword impressions from `directives/keyword_intelligence.md` data

---

## Edge Cases and Notes

- **Apify costs:** The Google Search Scraper runs all your ICP keywords × 2 pages per keyword per week. The Website Content Crawler adds cost for the content gap crawl. Facebook and LinkedIn ad scrapers cost approximately $2.50/week combined. LinkedIn Posts Scraper depends on company count. Estimate total for this module at approximately $4-6/week depending on keyword count.
- **Rate limiting:** Apify handles rate limiting internally. If an actor run fails, retry once after 60 seconds. If it fails again, log the error and continue with the data that was collected.
- **Content Crawler failures:** Some pages may block crawling (e.g., Cloudflare protection, login walls). If a page fails to crawl, log it as "blocked" and skip. The analysis layer works with whatever content was successfully crawled.
- **LinkedIn Posts Scraper - company discovery:** For competitors where only the company name is known (not the LinkedIn URL), the script should search for the company by name using the actor's search functionality. If multiple results are returned, select the one with the most followers or the verified company page. Cache discovered LinkedIn URLs in `.tmp/linkedin_company_urls.json` so they don't need to be re-discovered each week.
- **Competitor list changes:** If competitors are added or removed from `config/competitor_lists.py`, the script automatically picks them up on the next run. No code changes needed.
- **Duplicate competitors:** Some competitors may appear in multiple sub-vertical lists. The SERP tagging in Step 6 handles this by assigning multiple tags.
- **First run:** No previous snapshots exist for SERP, Meta ads, LinkedIn ads, or LinkedIn posts. Save current state and skip all change detection.
- **Brand keyword SERP results:** Brand keyword searches will mostly return the competitor's own website at position 1. The value is seeing if your comparison pages (e.g., "[Your Company] vs [Competitor]") appear in the results, and whether competitors are bidding on each other's brand terms.

---

## Scripts This Directive Feeds

- `execution/competitor_serp_scan.py` - Steps 1 through 6 (SERP scanning, Meta ad library, LinkedIn ad library, LinkedIn posts, website content crawling, and snapshot tracking)
- `execution/analyze.py` - Content gap classification (Step 5 analysis), LinkedIn posts analysis, and all reporting
