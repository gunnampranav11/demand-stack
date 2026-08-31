# Keyword Intelligence + Page 2 Recovery

## Goal
Every week, pull all keyword performance data from Google Ads, Google Search Console, and GA4. Score every keyword against the ICP taxonomy. Identify page 2 recovery opportunities (keywords ranking positions 11–20 with high impressions). Flag wasted spend on non-ICP keywords. Detect any new or reactivated campaigns automatically.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. Google Ads API**
- Account ID: `[YOUR_GOOGLE_ADS_ACCOUNT_ID]`
- Account type: Managed under MCC `[YOUR_MCC_ID]` (if applicable — remove if not using an MCC)
- Conversion actions to track: replace with your actual conversion action names from Google Ads

**2. Google Search Console API**
- Property: `https://[YOUR_DOMAIN]`

**3. GA4 Data API**
- Property ID: `[YOUR_GA4_PROPERTY_ID]`
- Active conversion events: replace with your actual GA4 conversion event names
  <!-- # Replace with your actual GA4 conversion event names -->
  <!-- Example: demo_request_submitted, trial_started, contact_form_submitted -->
- Ignore inactive/deprecated events: list any events you no longer track

---

## Step 1: Campaign Discovery (Run Every Time)

Before pulling keyword data, the script must discover the current state of all campaigns. Never hardcode campaign names.

**What to do:**
1. Call the Google Ads API and pull ALL campaigns in the account (active, paused, removed — everything)
2. For each campaign, record: campaign name, campaign ID, status (ENABLED / PAUSED / REMOVED), campaign type, budget, start date
3. Compare against the previous week's campaign list (stored in `.tmp/campaign_snapshot.json`)
4. Flag and report:
   - **New campaigns:** Any campaign ID that did not exist last week
   - **Reactivated campaigns:** Any campaign that was PAUSED last week and is now ENABLED
   - **Paused campaigns:** Any campaign that was ENABLED last week and is now PAUSED
   - **Budget changes:** Any campaign where the daily budget changed from last week
5. Save the current campaign list as the new `.tmp/campaign_snapshot.json` for next week's comparison
6. On first run (no previous snapshot exists), treat all ENABLED campaigns as current and save the snapshot without flagging changes

**Known active campaigns (for reference only — script discovers dynamically):**
<!-- Replace with your own campaign names as a reference. The script never relies on these. -->
- [Campaign 1 name]
- [Campaign 2 name]
- [Campaign 3 name]

---

## Step 2: Google Ads Keyword Pull

**What to pull for each ENABLED campaign:**
- Keyword text
- Match type (broad, phrase, exact)
- Campaign name
- Ad group name
- Impressions
- Clicks
- CTR
- Average CPC
- Cost (spend)
- Conversions (broken out by each conversion action)
- Conversion rate
- Quality Score (if available)
- Search impression share

**Filters:**
- Date range: last 7 days
- Include all ENABLED campaigns (not just the known ones — use the dynamic list from Step 1)
- Include keywords with 0 impressions (they may indicate paused keywords or match type issues)
- Also pull last 7 days of keyword data for any campaign flagged as newly paused in Step 1

**Save to:** `.tmp/google_ads_keywords.csv`

---

## Step 3: Google Search Console Pull

**What to pull:**
- Query (the search term)
- Page (the URL that ranked)
- Clicks
- Impressions
- CTR
- Average position

**Filters:**
- Property: `https://[YOUR_DOMAIN]`
- Date range: last 7 days
- All queries (no filters — pull everything)

**Page 2 recovery identification:**
After pulling all data, flag any query where:
- Average position is between 11.0 and 20.0 (page 2)
- Impressions are 50 or more in the 7-day window

These are recovery opportunities — keywords that are close to page 1 but not there yet.

**Save to:** `.tmp/gsc_keywords.csv`

---

## Step 4: GA4 Landing Page + Conversion Pull

**What to pull:**
- Landing page path
- Sessions
- Engaged sessions
- Bounce rate
- Conversions broken out by each active event
- Source / medium (to separate organic vs paid vs referral)

**Filters:**
- Property ID: `[YOUR_GA4_PROPERTY_ID]`
- Date range: last 7 days
- All landing pages (pull everything, do not filter)

**Save to:** `.tmp/ga4_keyword_landing_pages.csv`

---

## Step 5: ICP Scoring

Using the ICP keyword lists from `config/icp_taxonomy.py`, score every keyword from Google Ads and GSC:

**Scoring rules:**
- **ICP Match:** Keyword contains or closely matches a term from `ICP_KEYWORDS` in the taxonomy. Tag with the matching vertical.
- **Adjacent:** Keyword is industry-related but not a direct ICP term. Tag as adjacent.
- **Non-ICP:** Keyword has no relation to any ICP vertical. Tag as non-ICP.

**For Google Ads keywords specifically, also flag:**
- **Zero-conversion alerts:** Any keyword with more than $50 spend in the 7-day window and 0 conversions across all conversion actions
- **High-spend non-ICP:** Any non-ICP keyword with more than $25 spend
- **Low quality score alerts:** Any keyword with a quality score below 5

---

## Step 6: Cross-Source Matching

Match Google Ads keywords to GSC queries where possible:
- If a Google Ads keyword appears in GSC data, combine the data to show: paid spend + organic position + organic impressions for that term
- This reveals opportunities where you are paying for clicks on terms you could rank organically (or vice versa)

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/google_ads_keywords.csv` | All keyword data from Google Ads with ICP scores |
| `.tmp/gsc_keywords.csv` | All GSC queries with page 2 recovery flags |
| `.tmp/ga4_keyword_landing_pages.csv` | Landing page performance with conversion data |
| `.tmp/campaign_snapshot.json` | Current campaign states for next week's comparison |
| `.tmp/campaign_changes.json` | New, reactivated, paused, and budget-changed campaigns |

---

## What the Analysis Layer Does With This Data

The analysis layer (`execution/analyze.py` using Claude API) receives these CSV files and produces the following for Section 1.1 / 2.1 of the weekly report:

Per vertical (based on your ICP taxonomy):
- Top 10 performing ICP keywords by conversions
- Top 10 performing ICP keywords by impressions (awareness)
- Wasted spend summary: total dollars spent on non-ICP keywords
- Zero-conversion spend alerts with specific keyword + spend amounts
- Page 2 recovery opportunities ranked by impressions (highest first)
- Cross-source opportunities (paying for terms you rank well organically, or organic gaps where paid is needed)

Cross-vertical:
- Campaign change alerts (new, reactivated, paused, budget changes)
- Total ICP vs non-ICP spend ratio
- Week-over-week trend (requires previous week's data in `.tmp/`)

---

## Edge Cases and Notes

- **First run:** No previous snapshot exists. Save current state and skip campaign change detection. No week-over-week trends available.
- **API rate limits:** Google Ads API has a limit of 15,000 requests per day for basic access. A single weekly pull will use far less than this. If rate-limited, wait 60 seconds and retry.
- **GSC data delay:** GSC data is typically delayed by 2–3 days. The 7-day window accounts for this, but the most recent 2 days may show incomplete data. This is acceptable.
- **Zero-impression keywords:** Keep them in the output. They indicate paused keywords or match type issues that the analysis layer should surface.
- **Campaign names may change:** Never rely on campaign names for logic. Always use campaign IDs for tracking and comparison. Names are for display only.
- **GA4 event name spacing:** Some GA4 event names may have unusual spacing or formatting — use exact strings as configured in your GA4 property. Do not normalize.

---

## Scripts This Directive Feeds

- `execution/google_ads_pull.py` — Steps 1 and 2
- `execution/gsc_pull.py` — Step 3
- `execution/ga4_pull.py` — Step 4
- `execution/analyze.py` — Steps 5, 6, and the analysis layer
