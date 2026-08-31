# Email Sequence Performance

## Goal
Every week, pull all email sequence activity from HubSpot to track how outbound sales sequences are performing. Measure sends, opens, replies, and bounces per sequence and per step. Give priority attention to your primary ad-lead enrollment sequence. Identify which sequences and which email steps drive the most replies. Flag sequences with poor open or reply rates. All data is pulled from HubSpot's CRM API using EMAIL objects and contact sequence properties — no separate Sequences API endpoint is needed.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days for new email activity, plus rolling metrics on active sequences
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Source

**HubSpot API**
- Access: Private app token (stored in `.env` as `HUBSPOT_ACCESS_TOKEN`)
- Objects used: EMAIL (object type `0-49`), CONTACT
- Key EMAIL properties: `hs_sequence_id`, `hs_email_subject`, `hs_email_status`, `hs_email_open_count`, `hs_email_reply_count`, `hs_email_open_rate`, `hs_email_reply_rate`, `hs_email_direction`, `hs_timestamp`, `hs_email_to_email`, `hubspot_owner_id`
- Key CONTACT properties: `hs_sequences_is_enrolled`, `hs_latest_sequence_enrolled`, `hs_latest_sequence_enrolled_date`, `hs_sequences_enrolled_count`, `hs_latest_sequence_ended_date`

---

## Priority Sequence

**Sequence ID: `[YOUR_PRIORITY_SEQUENCE_ID]`**
<!-- Replace with your primary ad-lead enrollment sequence ID -->
<!-- Find sequence IDs in HubSpot: Automation → Sequences → click on a sequence → check the URL for the ID -->

This is your primary sequence — the one that all leads from paid search, LinkedIn, and Meta are enrolled in. It gets its own dedicated section in the weekly report, separate from the general sequence summary.

**Priority sequence reporting includes:**
- Total leads enrolled this week, broken out by original source (paid search, LinkedIn, Meta, organic, direct, referral)
- Per-step performance: sends, opens, open rate, replies, reply rate for each email step
- Overall open rate and reply rate
- Comparison against all other sequences (is the priority sequence performing better or worse?)
- Week-over-week trend for this sequence specifically
- Any step with a reply rate below 2% or open rate below 20% is flagged for rewrite

The priority sequence is always the first thing surfaced in the email sequence section of the weekly report.

---

## Step 1: Discover All Active Sequences

**What to do:**
Query EMAIL objects that have a `hs_sequence_id` to discover all sequences that have sent emails. The script should collect all unique `hs_sequence_id` values to build a dynamic list of sequences.

**How to discover:**
1. Query EMAIL objects with filter: `hs_sequence_id HAS_PROPERTY` (has any value)
2. Pull properties: `hs_sequence_id`, `hs_email_subject`
3. Group by `hs_sequence_id` to get a list of all unique sequence IDs
4. For each sequence ID, the email subjects will reveal the sequence name and step structure
5. Always ensure the priority sequence ID (`[YOUR_PRIORITY_SEQUENCE_ID]`) is included in the tracking list, even if it has fewer sends than other sequences

**Compare against previous week's snapshot (stored in `.tmp/email_sequence_snapshot.json`):**
- Flag any new sequence ID that didn't exist last week
- Flag any sequence that was active last week but has no new sends this week
- On first run, save snapshot and skip change detection

**Save to:** `.tmp/email_sequence_list.csv` and `.tmp/email_sequence_snapshot.json`

---

## Step 2: Pull All Sequence Emails (Last 7 Days)

**What to do:**
Query all EMAIL objects where `hs_sequence_id` has a value AND `hs_timestamp` is within the last 7 days.

**For each email, pull:**
- `hs_object_id` (email ID)
- `hs_sequence_id` (which sequence)
- `hs_email_subject` (subject line — reveals the step/template)
- `hs_email_status` (SENT, BOUNCED, FAILED, etc.)
- `hs_email_direction` (should be FORWARDED_EMAIL or EMAIL for outbound)
- `hs_email_open_count` (number of opens)
- `hs_email_reply_count` (number of replies)
- `hs_email_open_rate` (open rate — 100% if opened, 0% if not)
- `hs_email_reply_rate` (reply rate — 100% if replied, 0% if not)
- `hs_timestamp` (when sent)
- `hs_email_to_email` (recipient email)
- `hubspot_owner_id` (which rep sent it)

**Tag each email with `is_priority_sequence: true/false`** based on whether `hs_sequence_id` equals `[YOUR_PRIORITY_SEQUENCE_ID]`.

**Pagination:** The search endpoint returns a maximum of 200 results per page. The script must paginate through all results.

**Save to:** `.tmp/email_sequence_sends.csv`

---

## Step 3: Pull Contact Enrollment Data

**What to do:**
Query all contacts currently enrolled in sequences, plus contacts whose enrollment ended in the last 7 days.

**Pull 1: Currently enrolled contacts:**
- Filter: `hs_sequences_is_enrolled EQ true`
- Properties: `firstname`, `lastname`, `email`, `company`, `jobtitle`, `hs_sequences_is_enrolled`, `hs_latest_sequence_enrolled`, `hs_latest_sequence_enrolled_date`, `hs_sequences_enrolled_count`

**Pull 2: Recently completed/unenrolled contacts:**
- Filter: `hs_latest_sequence_ended_date GTE [7 days ago]`
- Properties: same as above, plus `hs_latest_sequence_ended_date`

**For each contact, also pull:**
- Original source (`hs_analytics_source` + `hs_analytics_source_data_1`) — to understand which channels feed into sequences
- Associated deal (if any) — to connect sequence activity to pipeline
- Lifecycle stage
- ICP tier (if available from `directives/attribution.md` outputs)

**Tag each contact with `is_priority_sequence: true/false`** based on whether `hs_latest_sequence_enrolled` equals `[YOUR_PRIORITY_SEQUENCE_ID]`.

**Save to:** `.tmp/email_sequence_contacts.csv`

---

## Step 4: Aggregate Metrics Per Sequence

**What to calculate for each sequence ID:**

**Volume metrics:**
- Total emails sent this week
- Total unique contacts emailed this week
- Total currently enrolled contacts
- Total contacts who completed/unenrolled this week

**Engagement metrics:**
- Open rate (emails with opens ÷ total emails sent)
- Reply rate (emails with replies ÷ total emails sent)
- Bounce count (emails with status BOUNCED)
- Bounce rate (bounced ÷ total sent)

**Per-step metrics (grouped by email subject within each sequence):**
- Each unique subject line represents a sequence step
- For each step: sends, opens, open rate, replies, reply rate
- This reveals which steps in the sequence are performing best and which are losing engagement

**Per-rep metrics (grouped by `hubspot_owner_id`):**
- For each rep: total sequence emails sent, open rate, reply rate

**Source metrics (from contact enrollment data):**
- For each original source channel: how many contacts from that source are enrolled in sequences
- Reply rate by source — do leads from paid search respond better than leads from LinkedIn?

**Priority sequence gets all of the above calculated separately** in addition to being included in the overall totals.

**Save to:** `.tmp/email_sequence_metrics.csv`

---

## Step 5: Week-Over-Week Comparison

**What to do:**
Compare this week's sequence metrics against last week's snapshot (stored in `.tmp/email_sequence_snapshot.json`).

**Flag and report:**
- **Volume change:** More or fewer sequence emails sent this week vs. last week
- **Open rate change:** Is the overall open rate improving or declining?
- **Reply rate change:** Is the overall reply rate improving or declining?
- **Priority sequence trend:** Separate week-over-week trend for the priority sequence
- **New sequences:** Any sequence ID that didn't exist last week
- **Inactive sequences:** Any sequence that sent emails last week but none this week
- **Step performance shifts:** Any email step where open or reply rate changed by more than 20%
- **Per-rep changes:** Any rep whose reply rate changed significantly

**Save current state as:** `.tmp/email_sequence_snapshot.json`
**Save changes as:** `.tmp/email_sequence_changes.json`

On first run, save snapshot and skip change detection.

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/email_sequence_list.csv` | All discovered sequences with IDs and email subjects |
| `.tmp/email_sequence_sends.csv` | All sequence emails from the last 7 days with engagement data and priority tag |
| `.tmp/email_sequence_contacts.csv` | All contacts currently enrolled or recently completed with priority tag |
| `.tmp/email_sequence_metrics.csv` | Aggregated metrics per sequence, per step, per rep, per source |
| `.tmp/email_sequence_snapshot.json` | Current sequence state for next week's comparison |
| `.tmp/email_sequence_changes.json` | Week-over-week changes in volume, engagement, and sequence activity |

---

## What the Analysis Layer Does With This Data

**Priority sequence section (appears first in Section 1.10 and 2.10):**
- **Dedicated dashboard:** Enrollment count this week by source (paid search, LinkedIn, Meta, organic, direct, referral)
- **Step-by-step breakdown:** Each email step's sends, open rate, and reply rate
- **Trend:** Week-over-week open rate and reply rate for this sequence specifically
- **Flagged steps:** Any step with reply rate below 2% or open rate below 20%, with recommendation to rewrite
- **Source comparison:** Which ad platforms' leads respond best to this sequence
- **Comparison:** How this sequence performs vs. all other active sequences

**General sequence section (follows priority sequence):**
- **Sequence health summary:** Total emails sent, overall open rate, overall reply rate, bounce rate across all sequences
- **Sequence-by-sequence breakdown:** Each active sequence's performance — sends, opens, replies, and trend vs. last week
- **Step-level analysis:** Within each sequence, which steps are performing well and which are losing engagement
- **Rep performance:** Which reps have the best reply rates on their sequences
- **Best performing email:** The specific email step with the highest reply rate this week
- **Worst performing email:** The specific email step with the lowest open rate — recommendation to rewrite or remove
- **Enrollment health:** How many contacts are currently enrolled, how many completed, is the pipeline growing or shrinking

**For Section 3 (Cross-Vertical Summary):**
- Priority sequence health: quick open/reply rate summary and trend
- Overall sequence engagement trend (reply rate up or down)
- Any sequence that needs immediate attention (very low reply rates or high bounce rates)

---

## Edge Cases and Notes

- **HubSpot API pagination:** EMAIL object queries may return thousands of results. The script must handle pagination using the `offset` parameter. Maximum 200 results per page.
- **HubSpot API rate limits:** Private apps allow 100 requests per 10 seconds. Include a small delay (0.2 seconds) between paginated requests.
- **Sequence names not directly available:** The HubSpot CRM API returns `hs_sequence_id` (a numeric ID) but not the human-readable sequence name. The script should build a mapping of sequence ID → name by analyzing the email subjects associated with each sequence.
- **Email step identification:** Steps within a sequence are identified by grouping emails with the same `hs_sequence_id` by subject line. If the same sequence has different subject lines, each unique subject represents a different step.
- **Open rate and reply rate values:** HubSpot stores these as 0 or 100 (not 0.0 to 1.0). A value of 100 means the email was opened/replied to; 0 means it was not. To calculate aggregate rates, average these values across all emails.
- **Bounce handling:** Emails with `hs_email_status` of BOUNCED should be counted separately. High bounce rates indicate list quality issues.
- **Owner ID mapping:** `hubspot_owner_id` returns a numeric ID. Use the HubSpot Owners API (`/crm/v3/owners`) to map owner IDs to rep names.
- **Third-party integration emails:** Some emails in HubSpot may be from integrations (e.g., Apollo, Outreach). Filter these out by only including emails where `hs_sequence_id` has a value. Emails without a `hs_sequence_id` will be naturally excluded.
- **Email direction:** Only include outbound emails (sent by reps to contacts). Filter by `hs_email_direction` to exclude incoming replies from the send count.
- **Dependency on other modules:** Contact source data benefits from `directives/attribution.md` having run first. The orchestrator should run attribution before this module.
- **First run:** No previous snapshot exists. Save current state and skip week-over-week comparison.

---

## Scripts This Directive Feeds

- `execution/email_sequence_pull.py` — Steps 1, 2, 3, 4, and 5
- `execution/analyze.py` — All reporting
