# GBP Competitive Analysis

## Goal
Every two weeks, scrape Google Business Profiles for your company and all major competitors. Analyze across 7 sub-modules: category and attributes audit, posts strategy, outlier identification, review framework, photo plan, services section optimization, and description optimization. Produce directly actionable recommendations - not generic SEO advice. Every recommendation must cite specific competitor evidence.

---

## When This Runs
- **Frequency:** Bi-weekly (every other Sunday as part of the main orchestrator)
- **Data window:** Current snapshot (GBP data is point-in-time)
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Source

**Google Maps Scraper**
- Actor ID: `compass/crawler-google-places`
- Pricing: $4.00 per 1,000 places on Starter plan

**Your GBP Details:**
<!-- Fill in your company's GBP details below -->
- Business name: `[YOUR_COMPANY_GBP_NAME]`
- Listed address: `[YOUR_GBP_ADDRESS]`
- Primary category: `[YOUR_GBP_CATEGORY]`
- Secondary categories: `[YOUR_GBP_SECONDARY_CATEGORIES]`
- GBP managed by: `[GBP_MANAGER_NAMES]`

---

## Step 1: Scrape All GBP Listings

**What to do:**
Run the Google Maps Scraper (`compass/crawler-google-places`) for your company and all competitors.

**Full search list (replace with your company and actual competitors):**
- [YOUR_COMPANY] [CITY]
- [YOUR_COMPANY]
- [COMPETITOR_1] [CITY]
- [COMPETITOR_2]
- [COMPETITOR_3]
- [COMPETITOR_4]
- [COMPETITOR_5]
- [add more competitors as needed]

**Settings:**
- Location: United States (adjust for your target market)
- Max results per search: 5 (to capture variations and secondary listings)
- Include reviews: ON
- Max reviews: Pull all reviews the actor returns. Do not set an artificial limit.

**For each listing found, record everything the actor returns. At minimum, expect these fields to be reliably available:**
- Business name
- Address
- Phone number
- Website URL
- Primary category
- Secondary categories (if returned by actor)
- Overall rating
- Total review count
- Reviews (text, rating, date, reviewer name, owner response if any)
- Business description (if the listing has one set)
- Services listed (names only)
- Total photo count
- Photo URLs (if returned by actor)
- Hours of operation
- Latitude / longitude

**Fields that may or may not be returned (handle gracefully):**
- **Posts:** Some Google Maps scrapers return GBP posts, some do not. If the actor returns post data, capture it (content, date, type). If not, log as "posts data unavailable" for that listing and skip.
- **Attributes** (accessibility, amenities, payment methods, etc.): May not be returned by the actor. If not available, skip the attributes portion of the category audit and note it.
- **Photo type classification** (interior, exterior, product, team, logo): The actor returns photo URLs and total count, but NOT category labels. Do not attempt to classify photos by type from URLs alone.

**Compare against previous scrape's snapshot (stored in `.tmp/gbp_snapshot.json`):**
- Flag any competitor that added or changed categories
- Flag any competitor with significant review count increase (more than 5 new reviews since last scrape)
- Flag any competitor that added new services
- Flag any competitor that changed their business description
- On first run, save snapshot and skip change detection

**Handling search misses:**
- Any search that returns no relevant results (no listing matching a known competitor domain from `config/competitor_lists.py`): Log as "no GBP listing found" for that competitor and continue. Not all competitors have GBP listings.

**Save to:** `.tmp/gbp_raw_data.csv` and `.tmp/gbp_snapshot.json`

---

## Step 2: Sub-Module Data Preparation

Organize the raw GBP data into structured formats for each of the 7 sub-modules. Each sub-module's data is saved separately so the analysis layer can process them independently.

**2a. Category & Attributes Data**
For each listing, extract:
- Primary category
- All secondary categories (if available)
- All attributes set (if available)
- Compare your company's categories against every competitor's

If attributes data is not returned by the actor, note this in the output and skip the attributes comparison. The category comparison still runs - categories are reliably returned.

**Save to:** `.tmp/gbp_categories.csv`

**2b. Posts Data**
For each listing, extract:
- All posts returned by the actor (ideally covering the last 60 days)
- Post content, date, type (update, offer, event, product)
- Post frequency (posts per week)

If the actor does not return post data, save an empty file with a header row and a note: "Posts data not available from scraper." The analysis layer will skip the posts strategy sub-module for this run and recommend manual review of competitor GBP posts instead.

**Save to:** `.tmp/gbp_posts.csv`

**2c. Outlier Data**
For each listing, extract:
- Total review count
- Average rating
- Primary category
- Number of photos (total count)
- Number of services listed
- Whether they have posts (if post data is available)

**Save to:** `.tmp/gbp_outlier_data.csv`

**2d. Review Data**
For each listing, extract:
- All reviews with full text, rating, date
- Owner responses (text and response time if calculable from review date vs. response date)
- Review velocity (reviews per month, calculated from review dates)

**Save to:** `.tmp/gbp_reviews.csv`

**2e. Photo Data**
For each listing, extract:
- Total photo count
- Photo URLs (if returned by actor)

Photo type classification (interior, exterior, product, team, logo) is NOT available from the scraper. The analysis layer will recommend photo types based on GBP best practices and competitor photo counts.

**Save to:** `.tmp/gbp_photos.csv`

**2f. Services Data**
For each listing, extract:
- All service names listed

Service descriptions are typically NOT returned by the scraper - only service names. The analysis layer will write recommended descriptions based on the service names competitors have listed and ICP keywords from `config/icp_taxonomy.py`.

**Save to:** `.tmp/gbp_services.csv`

**2g. Description Data**
For each listing, extract:
- Full business description text (if the listing has one)

Many listings do not have a description set. If a listing has no description, record as "No description set" in the output.

**Save to:** `.tmp/gbp_descriptions.csv`

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/gbp_raw_data.csv` | Complete GBP data for all searches |
| `.tmp/gbp_snapshot.json` | Current GBP state for next scrape's comparison |
| `.tmp/gbp_categories.csv` | Category and attribute data for all listings |
| `.tmp/gbp_posts.csv` | Post data for all listings (may be empty if actor doesn't return posts) |
| `.tmp/gbp_outlier_data.csv` | Ranking factor data for outlier identification |
| `.tmp/gbp_reviews.csv` | All reviews with full text and owner responses |
| `.tmp/gbp_photos.csv` | Photo counts and URLs for all listings |
| `.tmp/gbp_services.csv` | Service names for all listings |
| `.tmp/gbp_descriptions.csv` | Business descriptions for all listings |

---

## What the Analysis Layer Does With This Data

The analysis layer (`execution/analyze.py` using Claude API) processes each sub-module's data and produces the following for Section 1.5 and 2.5 of the weekly report. **Every recommendation must be specific and cite competitor evidence. No generic advice.**

**Sub-Module 1: Category & Attributes Audit**
- Compare your company's primary category against competitors
- Identify what categories competitors are using that you are not
- If attributes data is available: identify what attributes competitors have set that you have not
- Produce specific recommendations: "Change primary category from '[current category]' to '[recommended category]' - [N] of [total] competitors use this category"

**Sub-Module 2: Posts Strategy**
- If post data is available: analyze competitor posting frequency and content themes, identify which competitors post regularly vs. not at all
- If post data is not available: produce the posting calendar based on GBP best practices and ICP vertical relevance
- Produce an 8-week posting calendar with specific post topics, full draft copy, and recommended post type for each
- Posts must be directly relevant to your ICP verticals - not generic company updates
- **Output format:** Calendar format with week number, post date, topic, full copy, post type, target vertical

**Sub-Module 3: Outlier Identification**
- Identify competitors that rank disproportionately well despite having fewer reviews, fewer photos, or a weaker profile than expected
- **Output format:** Observations only. No advice, no recommendations. Just factual observations about which competitors are outliers and what profile characteristics they have.

**Sub-Module 4: Review Framework**
- Calculate review velocity for your company and each competitor (reviews per month)
- Set review velocity targets based on competitor benchmarks
- Extract common keyword themes from positive and negative reviews
- Analyze owner response patterns: which competitors respond, how quickly, what tone
- Produce review response templates for common review themes
- **Output format:** Velocity targets, keyword themes table, response templates

**Sub-Module 5: Photo Plan**
- Compare your company's photo count against competitors
- Produce an 8-week photo upload plan recommending photo types based on GBP best practices (exterior, interior, team, product screenshots, office environment, event photos)
- Include geotagging recommendations
- **Output format:** Calendar format with week number, photo type, description, geotagging instructions

**Sub-Module 6: Services Section Optimization**
- Compare your listed service names against competitors' listed service names
- Identify services competitors list that you do not
- Produce optimized service descriptions for each recommended service (written by the analysis layer since the scraper only returns service names, not descriptions)
- Descriptions must include ICP-relevant keywords from `config/icp_taxonomy.py`
- **Output format:** Table with service name, recommended description, target keywords, competitor evidence

**Sub-Module 7: Description Optimization**
- Analyze competitor business descriptions for length, keywords, structure, and value proposition
- Note which competitors have no description set
- Produce 3 versions of an optimized GBP description for A/B testing
- Each version should emphasize a different angle (e.g., version 1 = primary vertical focus, version 2 = broad product, version 3 = accuracy/speed proof points)
- Include ICP keywords naturally - no keyword stuffing
- **Output format:** 3 full description texts, each labeled with the angle and target vertical

**Top 7 Ranking Levers Table (required in every GBP report):**
The analysis layer must produce a table summarizing the top 7 ranking levers for your GBP, based on competitor evidence. **Strict table format - no prose, no bullet points.**

| Lever | Evidence from Competitors | Why It Matters |
|---|---|---|
| (specific lever) | (cite specific competitor data) | (explain ranking impact) |

---

## Edge Cases and Notes

- **Bi-weekly scheduling:** The orchestrator (`execution/main.py`) must track which Sundays to run this module. Use a simple toggle: check if the last GBP run was more than 12 days ago. If yes, run. If no, skip.
- **Apify costs:** Searches × 5 max results ≈ place lookups per run. At $4/1,000 places, costs approximately $0.40 per run plus review data. Estimate $1-2/run, or $0.50-1/week averaged over bi-weekly runs.
- **GBP listing not found:** Some competitors may not have a GBP listing. The script should filter results by matching the business website URL against known competitor domains from `config/competitor_lists.py`. If no match is found, log it as "no GBP listing found" and skip.
- **Your company may have multiple listings:** Include searches for both your company name with a city and your company name alone to capture both your US and any international office listings if applicable.
- **Scraper data limitations:** The Google Maps Scraper reliably returns categories, reviews, photo counts, service names, and descriptions. It may NOT return posts, attributes, photo type classifications, or service descriptions. Each sub-module is designed to handle missing data gracefully.
- **Review data privacy:** Do not include reviewer email addresses or personal information beyond what is publicly visible on the review platforms.
- **First run:** No previous snapshot exists. Save current state and skip change detection. All 7 sub-modules produce full analysis on first run.

---

## Scripts This Directive Feeds

- `execution/gbp_scrape.py` - Steps 1 and 2
- `execution/analyze.py` - All 7 sub-module analyses and ranking levers table
