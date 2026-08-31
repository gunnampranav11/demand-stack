# HubSpot Closed-Loop Attribution

## Goal
Every week, pull all deal and contact data from HubSpot across all pipelines. Track how leads move through lifecycle stages. Connect every deal back to its original source (organic search, paid search, LinkedIn, Meta, direct, referral, etc.) so the weekly report can show exactly which channels are producing pipeline and revenue vs. just generating leads.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Two pulls — last 7 days for new activity, plus all open deals regardless of date
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Source

**HubSpot API**
- Access: Private app token (stored in `.env` as `HUBSPOT_ACCESS_TOKEN`)
- Scopes required: `crm.objects.contacts.read`, `crm.objects.companies.read`, `crm.objects.deals.read`, `timeline`

---

## Step 1: Pipeline and Stage Discovery (Run Every Time)

Never hardcode pipeline or stage names. Discover dynamically each run.

**What to do:**
1. Call the HubSpot Deals Pipeline API and pull ALL pipelines and their stages
2. For each pipeline, record: pipeline ID, pipeline name, all stages with stage ID, stage name, and display order
3. Compare against the previous week's snapshot (stored in `.tmp/hubspot_pipeline_snapshot.json`)
4. Flag and report:
   - **New stages added:** Any stage ID that did not exist last week
   - **Stages removed:** Any stage ID that existed last week but is gone
   - **Stage renamed:** Same stage ID but different label
5. Save current state as `.tmp/hubspot_pipeline_snapshot.json`
6. On first run, save snapshot and skip change detection

**Known pipelines and stage IDs (for reference only — script discovers dynamically):**
<!-- Replace with your own pipeline and stage IDs. These are examples only. -->
<!-- The script uses IDs, not names, for all comparisons. -->
<!-- Find your stage IDs in HubSpot: Settings → CRM → Pipelines → Edit pipeline → inspect stage IDs via API -->
<!--
Pipeline: [Your Pipeline Name]
  Stage: [Stage Name 1] — ID: [replace with your stage ID]
  Stage: [Stage Name 2] — ID: [replace with your stage ID]
  Stage: Closed Won      — ID: [replace with your stage ID]
  Stage: Closed Lost     — ID: [replace with your stage ID]
-->

---

## Step 2: Pull All Open Deals

**What to pull for every deal that is NOT in a closed stage (Closed Won, Closed Lost, or equivalent):**
- Deal ID
- Deal name
- Deal amount
- Deal stage (name and internal ID)
- Pipeline (name and ID)
- Deal owner
- Create date
- Close date (expected)
- Last activity date
- Days in current stage
- Associated company name
- Associated company domain
- Associated company employee count
- Associated contact name(s)
- Associated contact email(s)
- Associated contact job title(s)

**Save to:** `.tmp/hubspot_open_deals.csv`

---

## Step 3: Pull Deals That Changed Stage in the Last 7 Days

**What to pull:**
- All deals where the stage changed within the last 7 days (use the deal property history API for `dealstage`)
- For each deal, record: deal ID, deal name, previous stage, new stage, date of change, deal amount, pipeline

This captures:
- New deals entering the pipeline (stage set for the first time)
- Deals advancing forward
- Deals going backward
- Deals closing (moved to Closed Won or Closed Lost)

**Save to:** `.tmp/hubspot_stage_changes.csv`

---

## Step 4: Pull Recently Won Deals (Last 7 Days)

**What to pull for every deal that moved to Closed Won in the last 7 days:**
- Everything from Step 2, plus:
- Deal amount (actual closed revenue)
- Time from create date to close date (sales cycle length in days)
- Original source (first touch attribution — see Step 6)
- All sources in the journey (multi-touch — see Step 6)

**Save to:** `.tmp/hubspot_won_deals.csv`

---

## Step 5: Pull All New Contacts Created in the Last 7 Days

**What to pull:**
- Contact ID
- First name, last name
- Email
- Company name
- Job title
- Lifecycle stage
- Lead status
- Create date
- Original source (`hs_analytics_source`)
- Original source drill-down 1 (`hs_analytics_source_data_1`)
- Original source drill-down 2 (`hs_analytics_source_data_2`)
- First conversion (the form or page that created the contact)
- Recent conversion (the most recent form or page interaction)
- Associated deal IDs (if any)

**Save to:** `.tmp/hubspot_new_contacts.csv`

---

## Step 6: Attribution Mapping

For every deal and contact, map the original source to a clean channel label:

| HubSpot Source Value | Clean Channel Label | Notes |
|---|---|---|
| `ORGANIC_SEARCH` | Organic Search | |
| `PAID_SEARCH` | Paid Search (Google Ads) | |
| `EMAIL_MARKETING` | Email | |
| `DIRECT_TRAFFIC` | Direct | |
| `OTHER_CAMPAIGNS` | Other Campaigns | Check `hs_analytics_source_data_1` for details |
| `OFFLINE` | Offline / Manual Entry | |
| `SOCIAL_MEDIA` | Social | Check `hs_analytics_source_data_1` for platform |
| `PAID_SOCIAL` | Paid Social | Check `hs_analytics_source_data_1` for platform |
| `REFERRALS` | Referral | Check `hs_analytics_source_data_1` for referring domain |
| `INTEGRATION` | Integration (e.g., Calendly) | Check `hs_object_source_label` + `hs_object_source_detail_1` |

---

## Step 7: ICP Scoring of New Contacts

Using `config/icp_taxonomy.py`, score every new contact:

**Scoring rules:**
- **Tier 1 (ICP match):** Job title matches target titles AND company employee count meets minimum threshold AND company industry or deal context matches a target vertical
- **Tier 2 (Adjacent):** Matches 2 of the 3 criteria above
- **Tier 3 (Weak fit):** Matches 1 of the 3 criteria
- **Unqualified:** Matches 0 — consumer, student, competitor, or no relevant use case

**Save the ICP tier as a column in:** `.tmp/hubspot_new_contacts.csv`

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/hubspot_open_deals.csv` | All open deals across all pipelines with company and contact details |
| `.tmp/hubspot_stage_changes.csv` | All deals that changed stage in the last 7 days |
| `.tmp/hubspot_won_deals.csv` | Deals that moved to Closed Won in the last 7 days |
| `.tmp/hubspot_new_contacts.csv` | New contacts with source attribution and ICP scores |
| `.tmp/hubspot_pipeline_snapshot.json` | Current pipeline state for next week's comparison |
| `.tmp/hubspot_pipeline_changes.json` | Any pipeline or stage structure changes detected |

---

## What the Analysis Layer Does With This Data

**For Section 1.6 and 2.6 of the weekly report (per vertical):**
- Source attribution breakdown: how many leads and how much pipeline came from each channel this week
- ICP quality by source: which channels produce Tier 1 leads vs. unqualified leads
- Pipeline velocity: average days in each stage, deals that are stuck (more than 14 days in same stage)
- Won deal analysis: revenue by source, average sales cycle, ICP profile of closed deals
- Stage movement summary: how many deals advanced, how many went backward, how many closed

**For Section 3 (Cross-Vertical Summary):**
- Total spend vs. pipeline generated by channel (when combined with ad spend data from keyword_intelligence and lead_scoring)
- Channel ROI: cost per Tier 1 lead, cost per deal, cost per dollar of pipeline
- Zero-pipeline source alerts: any channel with significant spend but zero deals created

---

## Edge Cases and Notes

- **HubSpot API pagination:** The deals and contacts endpoints return a maximum of 100 records per page. The script must handle pagination (using the `after` cursor) to pull all records.
- **Association API:** To get company and contact details for each deal, use the HubSpot Associations API (v4). Pull deal-to-company and deal-to-contact associations separately, then join.
- **Property history:** To detect stage changes, use the `propertiesWithHistory` parameter on the deals endpoint with `dealstage` included. This returns the full history of stage changes with timestamps.
- **Rate limits:** HubSpot private apps allow 100 requests per 10 seconds. Include a small delay between paginated requests (0.2 seconds).
- **Deal stage IDs:** Always query using internal stage IDs, never display labels. Labels can be renamed without changing the ID — using the ID is the only reliable approach.
- **Multiple pipelines:** If you have more than one pipeline (e.g., a qualification pipeline and a sales pipeline), pull from both. Track cross-pipeline movement.
- **Calendly or integration contacts:** If a contact's source is `INTEGRATION`, check `hs_object_source_label` to identify which integration created them.
- **First run:** No previous pipeline snapshot exists. Save current state and skip change detection.

---

## Scripts This Directive Feeds

- `execution/hubspot_pull.py` — Steps 1 through 6
- `execution/analyze.py` — Step 7 and the analysis layer
