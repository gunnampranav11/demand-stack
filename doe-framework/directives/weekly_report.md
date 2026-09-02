# Weekly Report Generation + Slack Delivery

## Goal
Every week, take all raw data collected by modules 1-11 and produce a single comprehensive PDF report with specific, data-backed analysis and action recommendations. Send the PDF to Slack [YOUR_SLACK_CHANNEL] tagging [TEAM_MEMBER_1], [TEAM_MEMBER_2], and [TEAM_MEMBER_3]. This module is the analysis layer - it reads CSV files from `.tmp/`, sends them to the Claude API for intelligent analysis, assembles the structured output, generates the PDF, and delivers it.

---

## When This Runs
- **Frequency:** Weekly (every Sunday at [YOUR_DELIVERY_TIME])
- **Runs after:** All other modules have completed. This is always the last module in the orchestrator.
- **Output:** Final PDF saved to `.tmp/weekly_report.pdf` and delivered to Slack

---

## Data Sources (All From `.tmp/`)

This module does not pull any data from external APIs. It reads the output files produced by all other modules:

**From `directives/keyword_intelligence.md`:**
- `.tmp/google_ads_keywords.csv`
- `.tmp/gsc_keywords.csv`
- `.tmp/ga4_keyword_landing_pages.csv`
- `.tmp/campaign_snapshot.json`
- `.tmp/campaign_changes.json`

**From `directives/attribution.md`:**
- `.tmp/hubspot_open_deals.csv`
- `.tmp/hubspot_stage_changes.csv`
- `.tmp/hubspot_won_deals.csv`
- `.tmp/hubspot_new_contacts.csv`
- `.tmp/hubspot_pipeline_changes.json`

**From `directives/lead_scoring.md`:**
- `.tmp/linkedin_campaign_performance.csv`
- `.tmp/linkedin_leads.csv`
- `.tmp/linkedin_campaign_changes.json`
- `.tmp/meta_campaign_performance.csv`
- `.tmp/meta_leads.csv`
- `.tmp/meta_campaign_changes.json`
- `.tmp/lead_hubspot_match.csv`

**From `directives/competitor_audit.md`:**
- `.tmp/serp_results.csv`
- `.tmp/serp_changes.json`
- `.tmp/competitor_meta_ads.csv`
- `.tmp/competitor_linkedin_ads.csv`
- `.tmp/competitor_linkedin_posts.csv`
- `.tmp/content_gap_pages.csv`

**From `directives/reddit_mining.md`:**
- `.tmp/reddit_subreddit_posts.csv`
- `.tmp/reddit_competitor_mentions.csv`

**From `directives/gbp_analysis.md` (bi-weekly - may not exist every week):**
- `.tmp/gbp_categories.csv`
- `.tmp/gbp_posts.csv`
- `.tmp/gbp_outlier_data.csv`
- `.tmp/gbp_reviews.csv`
- `.tmp/gbp_photos.csv`
- `.tmp/gbp_services.csv`
- `.tmp/gbp_descriptions.csv`

**From `directives/conversion_funnel.md`:**
- `.tmp/ga4_conversion_funnel.csv`
- `.tmp/flagged_landing_pages.csv`
- `.tmp/landing_page_content.csv`
- `.tmp/ad_landing_page_matches.csv`
- `.tmp/conversion_funnel_changes.json`

**From `directives/linkedin_organic.md`:**
- `.tmp/company_linkedin_page_metrics.csv`
- `.tmp/company_linkedin_posts.csv`
- `.tmp/thought_leader_posts.csv`
- `.tmp/linkedin_organic_changes.json`

**From `directives/meeting_booking_funnel.md`:**
- `.tmp/calendly_event_types.csv`
- `.tmp/calendly_events.csv`
- `.tmp/calendly_hubspot_match.csv`
- `.tmp/calendly_funnel_metrics.csv`
- `.tmp/calendly_changes.json`

**From `directives/email_sequence_performance.md`:**
- `.tmp/email_sequence_sends.csv`
- `.tmp/email_sequence_contacts.csv`
- `.tmp/email_sequence_metrics.csv`
- `.tmp/email_sequence_changes.json`

---

## Step 1: Data Validation

Before sending anything to Claude API, verify that input files exist and contain data.

**For each expected file:**
1. Check if the file exists in `.tmp/`
2. If it exists, check if it has more than just a header row (i.e., contains actual data)
3. Log the status of each file: `found`, `empty`, or `missing`

**Handling missing data:**
- If a file is `missing` or `empty`, the corresponding section of the report should say: "Data not available this week - [module name] did not produce output. Check the orchestrator logs for errors."
- GBP files (from `directives/gbp_analysis.md`) will be missing on non-GBP weeks (since it runs bi-weekly). This is expected - the report should say "GBP analysis runs bi-weekly. Next scheduled run: [date]."
- Never skip the entire report because one module failed. Produce the report with whatever data is available.

**Save validation log to:** `.tmp/data_validation_log.json`

---

## Step 2: Send Data to Claude API for Analysis

**How this works:**
The raw CSV data is too large to send in a single Claude API call. Break the analysis into logical chunks, one per report section. Each call sends the relevant CSV data and receives structured analysis back.

**Claude API configuration:**
- **Default model:** `claude-sonnet-4-6` (for most section analysis calls - good balance of speed, quality, and cost)
- **Executive summary and cross-vertical summary model:** `claude-opus-4-7` (most complex reasoning - use for Call 11 and the executive summary)
- **Max tokens:** 8192 per call
- **Temperature:** 0 for all data analysis calls (deterministic output). Omit temperature (let the model default) for the executive summary call - slight variation helps with quality prose.

**System prompt for all analysis calls:**
```
You are a senior demand generation analyst at [YOUR_COMPANY], a B2B SaaS [YOUR_PRODUCT_CATEGORY] company. You produce executive-level weekly analysis for the head of marketing and growth leadership.

Rules:
1. Every recommendation must be specific and cite data. Never give generic advice.
2. Use exact numbers - dollars, percentages, counts. No vague language.
3. When recommending actions, specify exactly what to change, on which platform, for which campaign/keyword/page.
4. Flag problems with urgency levels: CRITICAL (action needed this week), HIGH (action needed within 2 weeks), MEDIUM (monitor and act if trend continues), LOW (informational).
5. Compare against ICP taxonomy when scoring leads and keywords. Reference the specific vertical.
6. When data is missing or incomplete, say so explicitly. Do not fabricate analysis.
7. Keep each section concise - aim for 3-5 key insights per section, not exhaustive data dumps.
```

**Analysis calls (one per report section):**

**Call 1: Keyword Intelligence (Section 1.1 / 2.1)**
- Send: `google_ads_keywords.csv`, `gsc_keywords.csv`, `ga4_keyword_landing_pages.csv`, `campaign_changes.json`
- Ask for: Top ICP keywords by conversions per vertical, wasted spend on non-ICP keywords, zero-conversion alerts, page 2 recovery opportunities, campaign change alerts, week-over-week trend
- Also ask for: Keyword action recommendations - which keywords to pause (high spend, zero conversions, non-ICP), which to increase bids on (ICP match, high impressions, low position), which new keywords to add (based on GSC queries with high impressions that aren't in Google Ads)

**Call 2: Lead Scoring (Section 1.2 / 2.2)**
- Send: `linkedin_campaign_performance.csv`, `linkedin_leads.csv`, `linkedin_campaign_changes.json`, `meta_campaign_performance.csv`, `meta_leads.csv`, `meta_campaign_changes.json`, `lead_hubspot_match.csv`
- Ask for: Lead quality breakdown by campaign, cost per Tier 1 lead, named-company vs. interest targeting comparison, campaign change alerts, zero-ICP alerts, platform budget recommendations

**Call 3: Competitor Audit (Section 1.3 / 2.3)**
- Send: `serp_results.csv`, `serp_changes.json`, `competitor_meta_ads.csv`, `competitor_linkedin_ads.csv`, `competitor_linkedin_posts.csv`, `content_gap_pages.csv`
- Ask for: Competitor SERP movement, ad messaging analysis, organic messaging themes, content gap recommendations (what content to create, in what format, for which keywords)

**Call 4: Reddit + Review Sentiment (Section 1.4 / 2.4)**
- Send: `reddit_subreddit_posts.csv`, `reddit_competitor_mentions.csv`
- Ask for: Top pain points from Reddit, buyer language tracker, competitor mention summary, competitor sentiment shifts

**Call 5: GBP Analysis (Section 1.5 / 2.5) - only on bi-weekly runs**
- Send: `gbp_categories.csv`, `gbp_posts.csv`, `gbp_outlier_data.csv`, `gbp_reviews.csv`, `gbp_photos.csv`, `gbp_services.csv`, `gbp_descriptions.csv`
- Ask for: All 7 sub-module outputs as defined in `directives/gbp_analysis.md`

**Call 6: Attribution (Section 1.6 / 2.6)**
- Send: `hubspot_open_deals.csv`, `hubspot_stage_changes.csv`, `hubspot_won_deals.csv`, `hubspot_new_contacts.csv`, `hubspot_pipeline_changes.json`
- Ask for: Source attribution breakdown, ICP quality by source, pipeline velocity, won deal analysis, stage movement summary, channel ROI calculations

**Call 7: Conversion Funnel (Section 1.7 / 2.7)**
- Send: `ga4_conversion_funnel.csv`, `flagged_landing_pages.csv`, `landing_page_content.csv`, `ad_landing_page_matches.csv`, `conversion_funnel_changes.json`
- Ask for: Funnel visualization, top and worst converting pages, ad copy vs. landing page mismatch report with specific recommendations, source comparison, wasted paid spend on zero-conversion pages

**Call 8: LinkedIn Organic (Section 1.8 / 2.8)**
- Send: `company_linkedin_page_metrics.csv`, `company_linkedin_posts.csv`, `thought_leader_posts.csv`, `linkedin_organic_changes.json`, plus `competitor_linkedin_posts.csv` for benchmarking
- Ask for: Page health, post performance summary, content format analysis, thought leader leaderboard, competitive benchmark, content theme analysis, next week's content recommendations

**Call 9: Meeting Booking Funnel (Section 1.9 / 2.9)**
- Send: `calendly_event_types.csv`, `calendly_events.csv`, `calendly_hubspot_match.csv`, `calendly_funnel_metrics.csv`, `calendly_changes.json`
- Ask for: Funnel summary, rep performance, event type performance, source-to-meeting mapping, meeting-to-deal connection, no-show analysis, week-over-week trend

**Call 10: Email Sequence Performance (Section 1.10 / 2.10)**
- Send: `email_sequence_sends.csv`, `email_sequence_contacts.csv`, `email_sequence_metrics.csv`, `email_sequence_changes.json`
- Ask for: Priority sequence dedicated section first - enrollment by source, step-by-step performance, trend, flagged steps. Then general sequence summary, rep performance, best/worst emails, enrollment health.

**Call 11: Cross-Vertical Summary (Section 3)**
- Model: `claude-opus-4-7` (most complex reasoning)
- Send: Summaries from calls 1-10 (not raw CSVs - the structured output from each previous call)
- Ask for: Total spend vs. pipeline by vertical, budget shift recommendations, top 5 actions this week ranked by impact, zero-conversion spend alerts (>$50 spend, 0 conversions), competitor movement alerts, overall channel ROI

**Call 12: Entity Health (Section 4) - only on bi-monthly runs**
- Send: Entity check data from `directives/entity_optimization.md` (when available)
- Ask for: Schema markup status, knowledge graph presence, brand consistency

**Each call should return structured JSON** with:
```json
{
  "section_title": "...",
  "vertical": "[vertical name]",
  "insights": [
    {
      "finding": "...",
      "urgency": "CRITICAL|HIGH|MEDIUM|LOW",
      "recommendation": "...",
      "data_points": ["..."]
    }
  ],
  "tables": [...],
  "raw_text": "..."
}
```

**Save all analysis output to:** `.tmp/analysis_output.json`

---

## Step 3: Assemble Report Structure

**Organize the analysis output into the final report structure:**

```
[YOUR_COMPANY] WEEKLY INTELLIGENCE REPORT
[Day] [Date] [Time] [Timezone]

EXECUTIVE SUMMARY
  - Top 5 actions this week (from Cross-Vertical Summary, ranked by impact)
  - Critical alerts (anything with urgency CRITICAL)
  - Key wins (any positive trends or achievements)

SECTION 1: [VERTICAL 1 NAME]
  1.1  Keyword Scorecard + Page 2 Recovery
  1.2  Lead Quality Report
  1.3  Competitor Keyword & Ad Intel + Content Gap Analysis
  1.4  Reddit Pain-Point Mining + Review Sentiment
  1.5  GBP Analysis (bi-weekly - include or note "next scheduled run")
  1.6  HubSpot Closed-Loop Attribution
  1.7  Website Conversion Funnel + Landing Page Content Analysis
  1.8  LinkedIn Organic Performance
  1.9  Meeting Booking Funnel
  1.10 Email Sequence Performance

SECTION 2: [VERTICAL 2 NAME]
  (Same 11 sub-sections, for each additional vertical)

SECTION 3: CROSS-VERTICAL SUMMARY
  - Total spend vs pipeline by vertical
  - Budget shift recommendations
  - Top 5 actions this week (ranked by impact)
  - Zero-conversion spend alerts (>$50 spend, 0 conversions)
  - Competitor movement alerts
  - Channel ROI comparison

SECTION 4: ENTITY HEALTH (bi-monthly)
  - Schema markup status
  - Knowledge graph presence
  - Brand consistency across web

APPENDIX: DATA SOURCES
  - List of all data files used and their status (found/empty/missing)
  - Date range for each data source
  - Any errors or warnings from the orchestrator
```

---

## Step 4: Generate PDF

**What to do:**
Take the assembled report structure and generate a formatted, multi-page PDF.

**PDF formatting requirements:**
- Title page with report name, date, and "[YOUR_COMPANY] Confidential"
- Table of contents with page numbers
- Professional formatting: clean fonts, consistent headings, adequate margins
- Tables should be properly formatted (not raw text dumps)
- Use color coding for urgency: red for CRITICAL, orange for HIGH, yellow for MEDIUM, gray for LOW
- Charts or visualizations where appropriate (e.g., spend vs. pipeline bar chart, funnel visualization)
- Target length: 5-8 pages for the main report, plus appendix. Concise and scannable - not a data dump.

**PDF generation library:** Use `reportlab` or `weasyprint` (Python libraries for PDF generation). The script should be self-contained - no external template files needed.

**Save to:** `.tmp/weekly_report.pdf`

---

## Step 5: Slack Delivery

**What to do:**
Send the PDF to Slack channel [YOUR_SLACK_CHANNEL], tagging your team members.

**Slack details:**
- Bot token: stored in `.env` as `SLACK_BOT_TOKEN`
- Channel: `[YOUR_SLACK_CHANNEL]`
- People to tag:
  - [TEAM_MEMBER_1]: `[SLACK_USER_ID_1]`
  - [TEAM_MEMBER_2]: `[SLACK_USER_ID_2]`
  - [TEAM_MEMBER_3]: `[SLACK_USER_ID_3]`

**Message format:**
```
:bar_chart: *[YOUR_COMPANY] Weekly Intelligence Report - [Date]*

<@[SLACK_USER_ID_1]> <@[SLACK_USER_ID_2]> <@[SLACK_USER_ID_3]>

*Top 3 Actions This Week:*
1. [Most impactful action from cross-vertical summary]
2. [Second most impactful]
3. [Third most impactful]

*Critical Alerts:* [count] items need immediate attention

Full report attached below.
```

**Note:** Slack incoming webhooks cannot upload files. To attach the PDF, use the Slack API `files.upload` method (requires a Slack Bot Token with `files:write` scope) OR upload the PDF to a shared location and include a download link. The script should attempt `files.upload` first. If that fails (no bot token available), fall back to posting the message with a note: "PDF saved to `.tmp/weekly_report.pdf` - manual upload required."

**Save delivery status to:** `.tmp/slack_delivery_log.json`

---

## Step 6: Recommendations Engine

**This is the core analytical value of the automation.** Every analysis call in Step 2 should produce specific recommendations. The following categories of recommendations should appear in every weekly report:

**Keyword & Campaign Recommendations:**
- Pause keyword [X] - spent $[amount] with 0 conversions over [N] weeks
- Increase bid on keyword [X] - ICP match, position [N], needs boost to page 1
- Add keyword [X] to Google Ads - appearing in GSC with [N] impressions but not in paid campaigns
- Pause campaign [X] - zero ICP leads generated, $[amount] wasted
- Reactivate campaign [X] - it was paused but drove [N] Tier 1 leads before pausing

**Budget Recommendations:**
- Shift $[amount]/week from [platform A] to [platform B] - cost per Tier 1 lead is [X] on A vs [Y] on B
- Shift budget from interest targeting to named-company targeting on LinkedIn - [data comparison]
- Reduce spend on non-ICP keywords by $[amount]/week

**Content Recommendations:**
- Create [content type] for keyword [X] - top 3 ranking pages are [blog posts / comparison pages / etc.] and your company has no comparable content
- Rewrite landing page [URL] - ad copy promises [X] but landing page talks about [Y]
- Publish LinkedIn post about [topic] - competitor [name] got [N] reactions on this theme

**Pipeline Recommendations:**
- Follow up on deal [X] - stuck in stage [Y] for [N] days
- Review lead source [X] - generating leads but zero pipeline
- Check Calendly no-show pattern on [day/time] - [N]% no-show rate

**GA4 / Tracking Recommendations:**
- Investigate GA4 event [X] - no stream data detected, tracking may be broken
- Review conversion event setup - [event name] has unusual behavior

**Each recommendation must include:**
- What to do (specific action)
- Why (data that supports it)
- Urgency level (CRITICAL / HIGH / MEDIUM / LOW)
- Which platform or tool to take action in

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/data_validation_log.json` | Status of all input files (found/empty/missing) |
| `.tmp/analysis_output.json` | All Claude API analysis output in structured JSON |
| `.tmp/weekly_report.pdf` | The final formatted PDF report |
| `.tmp/slack_delivery_log.json` | Slack delivery status and any errors |

---

## Edge Cases and Notes

- **Claude API token limits:** Each analysis call sends CSV data as text. Large CSVs may need to be truncated or summarized before sending. If a CSV exceeds 50,000 characters, the script should: (a) send the first 100 rows with a note about total row count, or (b) pre-aggregate the data into a summary table before sending. The analysis quality depends on seeing real data, not just summaries - so prefer sending actual rows when possible.
- **Claude API rate limits:** The Anthropic API has rate limits based on your plan. With 13 analysis calls per weekly run, this should be well within limits. If rate-limited, wait 60 seconds and retry.
- **Claude API cost:** Approximately 13 calls × ~3,000 input tokens + ~8,000 output tokens each ≈ ~143,000 tokens per run. At current `claude-sonnet-4-6` pricing, this is approximately $0.50-0.80 per weekly run. Monthly cost: ~$2-4 for the analysis layer alone. The two `claude-opus-4-7` calls (cross-vertical summary + executive summary) add a small premium.
- **PDF generation:** If `reportlab` fails, fall back to `weasyprint`. If both fail, fall back to generating a well-formatted Markdown file and converting to PDF using `pandoc` (if installed) or deliver the Markdown directly to Slack.
- **Slack file upload:** The incoming webhook cannot upload files. The script needs either a Slack Bot Token with `files:write` scope OR a workaround (upload to a shared drive and link). Document which approach is used during testing.
- **Partial data:** The report must be generated even if some modules failed. Each section should gracefully handle missing data with a clear note rather than crashing the entire report.
- **Bi-weekly modules:** GBP analysis (from `directives/gbp_analysis.md`) and entity health (from `directives/entity_optimization.md`) only run every other week or less. When their data is not available, the report should note: "GBP analysis runs bi-weekly. Last run: [date]. Next run: [date]." Do not show stale data from the previous bi-weekly run.
- **Report date and timezone:** The report header should show the date in your local timezone. All data ranges should be clearly stated (e.g., "Data: May 12-18, 2026").
- **First run:** Some week-over-week comparisons will not be available on the first run. The report should note: "Week-over-week comparison not available - this is the first run. Baseline data has been saved for next week's comparison."
- **Executive summary:** The executive summary at the top of the report is the most important section. It should be written last (after all other analysis is complete) using the cross-vertical summary data. A busy executive should be able to read just the executive summary and know the 3-5 most important things.

---

## Scripts This Directive Feeds

- `execution/analyze.py` - Steps 1, 2, and 6 (data validation, Claude API analysis, recommendations engine)
- `execution/generate_report.py` - Steps 3 and 4 (report assembly and PDF generation)
- `execution/slack_delivery.py` - Step 5 (Slack delivery)
