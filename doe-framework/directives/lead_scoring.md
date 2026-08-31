# Ad Platform Lead Scoring

## Goal
Every week, pull all lead and campaign performance data from LinkedIn Ads, Meta Ads, and cross-reference with HubSpot contacts. Score every lead against the ICP taxonomy. Flag campaigns that are generating high volumes of non-ICP leads. Detect any new or reactivated campaigns on both platforms automatically.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. LinkedIn Marketing API**
- Account ID: `[YOUR_LINKEDIN_ACCOUNT_ID]`

**2. Meta Marketing API**
- Ad Account ID: `[YOUR_META_AD_ACCOUNT_ID]`

**3. HubSpot API**
- Access: Private app token (stored in `.env` as `HUBSPOT_ACCESS_TOKEN`)
- Used for: Cross-referencing leads from LinkedIn and Meta against CRM contact records

---

## Step 1: LinkedIn Campaign Discovery (Run Every Time)

Never hardcode campaign names. Discover dynamically each run.

**What to do:**
1. Call the LinkedIn Campaign Manager API and pull ALL campaign groups and campaigns in your account (active, paused, archived — everything)
2. For each campaign group, record: campaign group ID, campaign group name, status
3. For each campaign within each group, record: campaign ID, campaign name, status, campaign type, objective, budget, start date, end date
4. Compare against the previous week's snapshot (stored in `.tmp/linkedin_campaign_snapshot.json`)
5. Flag and report:
   - **New campaigns:** Any campaign ID that did not exist last week
   - **Reactivated campaigns:** Any campaign that was PAUSED last week and is now ACTIVE
   - **Paused campaigns:** Any campaign that was ACTIVE last week and is now PAUSED
   - **Budget changes:** Any campaign where budget changed from last week
6. Save current state as `.tmp/linkedin_campaign_snapshot.json`
7. On first run, save snapshot and skip change detection

**Known active campaign groups (for reference only — script discovers dynamically):**
<!-- Replace with your own campaign group names as reference documentation -->
- [Campaign Group 1]
- [Campaign Group 2]

---

## Step 2: LinkedIn Campaign Performance Pull

**What to pull for each ACTIVE campaign:**
- Campaign group name
- Campaign name
- Campaign ID
- Campaign type / objective
- Impressions
- Clicks
- CTR
- Total spend
- Average CPC
- Average CPM
- Conversions (all types)
- Cost per conversion
- Lead gen form fills (if lead gen campaign)
- Video views and completion rate (if video campaign)

**Also pull last 7 days of performance data for any campaign flagged as "newly paused" in Step 1.** This captures any spend or leads that occurred before the campaign was paused this week.

**Filters:**
- Date range: last 7 days
- Include all ACTIVE campaigns plus any newly paused campaigns (use dynamic list from Step 1)

**Save to:** `.tmp/linkedin_campaign_performance.csv`

---

## Step 3: LinkedIn Lead Gen Form Pull

**What to pull for every lead gen form submission in the last 7 days:**
- Form name
- Campaign name that generated the submission
- Submission date
- First name
- Last name
- Email
- Company name
- Job title
- Any other fields captured by the form

**Note:** LinkedIn lead gen forms capture name, email, and profile data (company, job title) from the user's LinkedIn profile — not from form fields. The API returns profile data with each submission.

**Known lead gen forms (for reference only — script discovers dynamically):**
<!-- Replace with your own lead gen form names as documentation -->
- [Form 1 name]
- [Form 2 name]

**Save to:** `.tmp/linkedin_leads.csv`

---

## Step 4: Meta Campaign Discovery (Run Every Time)

Same approach as LinkedIn — never hardcode.

**What to do:**
1. Call the Meta Marketing API and pull ALL campaigns in your ad account (active, paused, archived — everything)
2. For each campaign, record: campaign ID, campaign name, status, objective, daily/lifetime budget
3. Also pull ad sets within each campaign: ad set ID, ad set name, status, targeting summary, budget
4. Compare against the previous week's snapshot (stored in `.tmp/meta_campaign_snapshot.json`)
5. Flag and report: new, reactivated, paused, budget changes (same logic as Step 1)
6. Save current state as `.tmp/meta_campaign_snapshot.json`
7. On first run, save snapshot and skip change detection

**Known Meta campaigns (for reference only — script discovers dynamically):**
<!-- Replace with your own campaign names as documentation -->
- [Campaign 1 name]
- [Campaign 2 name]

---

## Step 5: Meta Campaign Performance Pull

**What to pull for each ACTIVE campaign:**
- Campaign name
- Campaign ID
- Objective
- Impressions
- Reach
- Clicks (all)
- Link clicks
- CTR
- Total spend
- CPC
- CPM
- Conversions (all types)
- Cost per conversion
- Leads (if lead gen objective)
- Cost per lead

**Also pull at the ad set level:**
- Ad set name
- Ad set targeting summary (audiences, locations, demographics)
- Ad set spend
- Ad set conversions

**Also pull last 7 days of performance data for any campaign flagged as "newly paused" in Step 4.**

**Save to:** `.tmp/meta_campaign_performance.csv`

---

## Step 6: Meta Lead Pull

**What to pull for every lead generated in the last 7 days (if using Meta Lead Ads):**
- Campaign name
- Ad set name
- Submission date
- First name
- Last name
- Email
- Company name (if captured)
- Job title (if captured)

**Note:** Meta lead forms may capture fewer fields than LinkedIn. If company name or job title is not captured, leave those columns blank — the HubSpot cross-reference in Step 7 may fill them in.

**Save to:** `.tmp/meta_leads.csv`

---

## Step 7: HubSpot Cross-Reference

For every lead from LinkedIn (Step 3) and Meta (Step 6), look up the contact in HubSpot by email address.

**What to pull from HubSpot for each matched contact:**
- HubSpot contact ID
- Lifecycle stage
- Lead status
- Associated deal (if any) — deal ID, deal name, deal stage, deal amount
- Company name (from HubSpot, which may be more complete than the ad platform)
- Company employee count
- Company industry
- Original source (from HubSpot — to see if this contact existed before the ad interaction)

**What to flag:**
- **Already in CRM:** Lead was already a HubSpot contact before this week's ad interaction — this is a retargeting touch, not a net-new lead
- **New to CRM:** Lead did not exist in HubSpot before — this is a genuine new lead from the ad platform
- **Has deal:** Lead already has an associated deal — ad touch is influencing an existing opportunity
- **No match:** Lead email not found in HubSpot — possible data lag (integration sync delay)

**Save to:** `.tmp/lead_hubspot_match.csv`

---

## Step 8: ICP Scoring of All Leads

Using `config/icp_taxonomy.py`, score every lead from LinkedIn and Meta:

**Scoring rules:**
- **Tier 1 (ICP match):** Job title matches target titles AND company employee count meets minimum threshold AND company industry or context matches a target vertical
- **Tier 2 (Adjacent):** Matches 2 of the 3 criteria above
- **Tier 3 (Weak fit):** Matches 1 of the 3 criteria
- **Unqualified:** Matches 0 — consumer, student, competitor, no relevant use case

**Use the best data available:** If HubSpot has company employee count and the ad platform doesn't, use HubSpot's data. If the ad platform captured a job title but HubSpot doesn't have one, use the ad platform's data.

**Per-campaign ICP summary:**
For each campaign on both platforms, calculate:
- Total leads generated
- Count and percentage by tier (Tier 1, Tier 2, Tier 3, Unqualified)
- Cost per Tier 1 lead (campaign spend ÷ Tier 1 leads)
- Cost per qualified lead (campaign spend ÷ (Tier 1 + Tier 2) leads)

**Save ICP tier as a column in:** `.tmp/linkedin_leads.csv` and `.tmp/meta_leads.csv`

---

## Step 9: Named-Company vs. Interest/Group Targeting Analysis

**Why this matters:** Campaigns targeting named companies tend to outperform campaigns using interest-based or group targeting for B2B because you control exactly which accounts see your ads.

**What to do:**
For each LinkedIn campaign, determine the targeting type from the campaign data:
- **Named-company targeting:** Campaign targets a specific list of companies
- **Interest/group targeting:** Campaign targets LinkedIn groups, interests, or job functions broadly
- **Mixed:** Campaign uses some company targeting combined with broader targeting

**Flag and report:**
- For each targeting type, show: total spend, total leads, Tier 1 leads, cost per Tier 1 lead
- If an interest/group targeted campaign has zero Tier 1 leads and more than $200 spend, flag it as a zero-ICP alert

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/linkedin_campaign_performance.csv` | All LinkedIn campaign metrics |
| `.tmp/linkedin_leads.csv` | All LinkedIn lead gen form submissions with ICP scores |
| `.tmp/linkedin_campaign_snapshot.json` | Current LinkedIn campaign states for next week's comparison |
| `.tmp/linkedin_campaign_changes.json` | New, reactivated, paused, budget-changed LinkedIn campaigns |
| `.tmp/meta_campaign_performance.csv` | All Meta campaign metrics |
| `.tmp/meta_leads.csv` | All Meta leads with ICP scores |
| `.tmp/meta_campaign_snapshot.json` | Current Meta campaign states for next week's comparison |
| `.tmp/meta_campaign_changes.json` | New, reactivated, paused, budget-changed Meta campaigns |
| `.tmp/lead_hubspot_match.csv` | HubSpot cross-reference results for all leads |

---

## What the Analysis Layer Does With This Data

**For Section 1.2 and 2.2 of the weekly report (per vertical):**
- Lead quality breakdown by campaign: Tier 1 / Tier 2 / Tier 3 / Unqualified counts and percentages
- Cost per Tier 1 lead by campaign
- Best and worst performing campaigns by ICP lead quality
- New vs. already-in-CRM breakdown (are ads generating net-new pipeline or retouching known contacts?)
- Named-company vs. interest targeting performance comparison (LinkedIn)

**For Section 3 (Cross-Vertical Summary):**
- Total LinkedIn spend vs. Meta spend vs. Tier 1 leads generated on each
- Campaign change alerts (new, reactivated, paused, budget changes on both platforms)
- Zero-ICP campaign alerts: campaigns with significant spend but zero Tier 1 or Tier 2 leads
- Platform-level recommendation: where to shift budget based on cost-per-qualified-lead

---

## Edge Cases and Notes

- **LinkedIn API rate limits:** 100 requests per day per user for most endpoints. The script must batch requests efficiently. If rate-limited, log the error and retry after the cooldown period.
- **LinkedIn lead gen form data retention:** LinkedIn retains lead gen form submissions for 90 days. The script pulls only the last 7 days, which is well within the window.
- **Meta API versioning:** Use the latest stable Graph API version. Check for deprecation warnings in API responses and update the version as needed.
- **Meta lead form sync delay:** Meta leads may take up to 24 hours to appear in the API. The 7-day window accounts for this.
- **HubSpot cross-reference miss:** If a lead's email is not found in HubSpot, it may be due to sync delay. Log these as "No match" and include them in the lead count — they should resolve by next week.
- **Thought leader vs. paid:** If you track organic LinkedIn posts from individuals separately, scope this directive to paid campaigns only.
- **First run:** No previous snapshots exist. Save current state and skip campaign change detection for both platforms.

---

## Scripts This Directive Feeds

- `execution/linkedin_pull.py` — Steps 1, 2, 3, and part of Step 9
- `execution/meta_pull.py` — Steps 4, 5, 6
- `execution/hubspot_pull.py` — Step 7 (extends the same script from `directives/attribution.md` to support email lookup)
- `execution/analyze.py` — Steps 8, 9, and the analysis layer
