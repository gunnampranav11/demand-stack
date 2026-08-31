# Meeting Booking Funnel

## Goal
Every week, pull all Calendly meeting data to track the full booking funnel — from scheduled to completed to no-show. Cross-reference with HubSpot to connect meetings to deals, lead sources, and meeting outcomes. Identify which event types, which reps, and which lead sources produce the most completed meetings. Surface no-show patterns and booking drop-offs.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days for new bookings, plus status updates on meetings booked in prior weeks
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. Calendly API**
- Access: API token (stored in `.env` as `CALENDLY_API_TOKEN`)
- Organization: your Calendly organization slug (find in Calendly → Account → Organization)
- Scopes available: `scheduled_events:read`, `event_types:read`, `organizations:read`, `users:read`, `activity_log:read`, `outgoing_communications:read`

**2. HubSpot API**
- Access: Private app token (stored in `.env` as `HUBSPOT_ACCESS_TOKEN`)
- Used for: Cross-referencing meeting invitees with CRM contacts and deals, and checking meeting outcomes
- Note: If your Calendly-HubSpot integration is active, Calendly bookings automatically create meeting events and contacts in HubSpot. Calendly-originated meetings in HubSpot typically have titles prefixed by the meeting type (e.g., "Calendly: [Event Type Name]").

---

## Step 1: Get Organization and User URIs

**What to do:**
Before pulling any data, the script must get the organization URI and all user URIs. These are required parameters for all subsequent Calendly API calls.

1. Call `GET /users/me` to get the current user's URI and organization URI
2. Call `GET /organization_memberships` with the organization URI to get all team members' user URIs
3. Store these URIs for use in subsequent steps

**This step is essential — Calendly API endpoints require `user` or `organization` URI parameters. Without them, all other calls will fail.**

---

## Step 2: Pull All Event Types

**What to do:**
Call the Calendly API and pull all event types across the organization. This discovers all meeting types dynamically — never hardcode event type names.

**For each event type, record:**
- Event type name
- Event type slug/ID
- Duration (minutes)
- Active or inactive status
- Owner (which team member)
- Whether it's an individual or shared/round-robin event type

**Known event types (for reference only — script discovers dynamically):**
<!-- Replace with your own event types as documentation. The script discovers these dynamically. -->
<!-- Example format: -->
<!-- - [Rep Name]: 30 minute meeting, schedule a demo, pricing call -->
<!-- - Shared: speak with our [product] team, intro call -->

**Save to:** `.tmp/calendly_event_types.csv`

---

## Step 3: Pull All Scheduled Events (Last 7 Days)

**What to do:**
Call the Calendly API and pull all scheduled events (meetings) created or occurring in the last 7 days. Use the organization URI from Step 1 to get events across all team members.

**For each event, record:**
- Event ID / URI
- Event type name
- Event status from Calendly: `active` or `canceled`
- Scheduled date and time
- Duration (minutes)
- Host name (the rep)
- Created at timestamp (when the booking was made)
- Cancellation reason (if canceled — available in the cancellation object)
- UTM parameters (if captured — utm_source, utm_medium, utm_campaign can reveal which ad or page drove the booking)

**For each event, also pull the invitee details** (via the invitees endpoint for each event):
- Invitee name
- Invitee email
- Questions and answers from the booking form
- No-show status (check via the no-show endpoint for each invitee)

**Determining meeting completion status:**
Calendly's API does NOT have a "completed" status. Events are either `active` or `canceled`. To determine if a meeting actually happened, use this logic:

- **Completed:** Event status is `active` AND event start time is in the past AND invitee is NOT marked as no-show
- **No-show:** Event status is `active` AND event start time is in the past AND invitee IS marked as no-show
- **Upcoming:** Event status is `active` AND event start time is in the future
- **Canceled:** Event status is `canceled`
- **Rescheduled:** Event status is `canceled` AND a new event exists with the same invitee email. Tag as "rescheduled" rather than "canceled" to avoid double-counting.

**Also pull events from the previous 7–14 days that may have had status updates** (e.g., a meeting booked 10 days ago that was marked as no-show this week). This ensures status changes are captured even for meetings booked in prior weeks.

**Save to:** `.tmp/calendly_events.csv`

---

## Step 4: HubSpot Cross-Reference

**What to do:**
For each Calendly invitee, look up the contact in HubSpot by email address.

**What to pull from HubSpot for each matched contact:**
- HubSpot contact ID
- Lifecycle stage
- Lead status
- Original source (`hs_analytics_source` + drill-downs)
- Associated deal (if any) — deal ID, deal name, deal stage, deal amount, pipeline
- Company name (from HubSpot)
- Company employee count
- Company industry
- ICP tier (if already scored by `directives/attribution.md` or `directives/lead_scoring.md` — check `.tmp/hubspot_new_contacts.csv` or `.tmp/linkedin_leads.csv` / `.tmp/meta_leads.csv`)

**Also check HubSpot meeting outcome as a supplementary data source:**
Search HubSpot MEETING_EVENT objects for meetings with titles starting with "Calendly:" that match the event's time and invitee. HubSpot tracks `hs_meeting_outcome` values including `COMPLETED` and `SCHEDULED`. If the Calendly-inferred status and HubSpot outcome disagree, prefer HubSpot's `hs_meeting_outcome` since it may have been manually updated by the rep.

**What to flag:**
- **Has deal:** Meeting invitee has an associated deal — this meeting is part of an active sales process
- **No deal:** Meeting invitee is in HubSpot but has no deal — potential new opportunity
- **New to CRM:** Meeting invitee's email is not in HubSpot — may indicate a sync issue
- **Meeting source:** Map the lead's original source to understand which channel drove the meeting

**Save to:** `.tmp/calendly_hubspot_match.csv`

---

## Step 5: Funnel Metrics Calculation

**What to calculate:**

**Overall funnel (last 7 days):**
- Total meetings scheduled
- Total meetings completed
- Total meetings canceled
- Total no-shows
- Total rescheduled
- Completion rate (completed ÷ (completed + no-shows + canceled))
- No-show rate (no-shows ÷ (completed + no-shows))
- Cancellation rate (canceled ÷ scheduled)

**Per rep:**
For each team member (host), calculate:
- Meetings scheduled
- Meetings completed
- No-shows
- Completion rate
- No-show rate

**Per event type:**
For each event type, calculate:
- Meetings scheduled
- Meetings completed
- No-shows
- Completion rate
- Which event types have the highest no-show rate

**Per lead source (from HubSpot cross-reference):**
For each original source channel, calculate:
- Meetings booked
- Meetings completed
- No-show rate per source
- Whether certain channels produce higher no-show rates

**Save to:** `.tmp/calendly_funnel_metrics.csv`

---

## Step 6: Week-Over-Week Comparison

**What to do:**
Compare this week's funnel metrics against last week's snapshot (stored in `.tmp/calendly_snapshot.json`).

**Flag and report:**
- **Volume change:** More or fewer meetings booked this week vs. last week
- **No-show rate change:** Is the no-show rate improving or worsening?
- **Rep activity change:** Any rep with significantly more or fewer meetings than last week
- **New event types:** Any event type that didn't exist last week
- **Removed event types:** Any event type that existed last week but is now inactive

**Save current state as:** `.tmp/calendly_snapshot.json`
**Save changes as:** `.tmp/calendly_changes.json`

On first run, save snapshot and skip change detection.

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/calendly_event_types.csv` | All event types across the organization |
| `.tmp/calendly_events.csv` | All events from the last 7–14 days with status, invitee details, and completion status |
| `.tmp/calendly_hubspot_match.csv` | HubSpot cross-reference for each invitee |
| `.tmp/calendly_funnel_metrics.csv` | Funnel metrics: overall, per rep, per event type, per source |
| `.tmp/calendly_snapshot.json` | Current funnel state for next week's comparison |
| `.tmp/calendly_changes.json` | Week-over-week changes in volume, no-shows, rep activity |

---

## What the Analysis Layer Does With This Data

**For Section 1.9 and 2.9 of the weekly report:**
- **Funnel summary:** Meetings scheduled → completed → no-show → canceled → rescheduled, with rates for each
- **Rep performance:** Which reps had the most completed meetings, which had the highest no-show rates
- **Event type performance:** Which meeting types drive the most bookings? Which have the worst no-show rates?
- **Source-to-meeting mapping:** Which channels produce meetings? Breakdown by paid search, organic, LinkedIn, Meta, direct, referral
- **Meeting-to-deal connection:** How many meetings this week were tied to existing deals vs. net-new?
- **No-show analysis:** Specific patterns — are certain days/times worse for no-shows? Are certain sources worse?
- **Week-over-week trend:** Is meeting volume growing or shrinking? Is completion rate improving?

**For Section 3 (Cross-Vertical Summary):**
- Total meetings booked vs. completed this week
- Meeting-to-deal conversion rate
- Top source driving completed meetings
- No-show rate trend

---

## Edge Cases and Notes

- **Calendly API requires user/organization URIs:** Every Calendly API call (except `GET /users/me`) requires either a `user` or `organization` URI parameter. Step 1 must run first to retrieve these. If Step 1 fails, the entire module should log the error and skip — do not proceed with hardcoded URIs.
- **Calendly API pagination:** The scheduled events endpoint paginates. The script must handle pagination using the `next_token` or `pagination` object returned in each response. Continue pulling until all events are retrieved.
- **Calendly API rate limits:** Calendly's API has rate limits (varies by plan). If rate-limited, wait and retry. For weekly pulls of a typical B2B org's meeting volume, this should not be an issue.
- **Organization-wide access:** The API token should have `organizations:read` scope, which returns events across all team members — not just the token owner's events. Verify this during testing.
- **Calendly does NOT have a "completed" event status:** The API only supports `active` and `canceled`. Completion must be inferred from event timing and no-show status (see Step 3). HubSpot's `hs_meeting_outcome` field provides a supplementary check.
- **No-show marking:** No-shows must be manually marked in Calendly by the host (or automatically if configured). If hosts don't consistently mark no-shows, the no-show data will be incomplete. The analysis layer should note this limitation.
- **Rescheduled meetings:** Calendly treats rescheduled meetings as a cancellation of the original + creation of a new event. The script should detect rescheduled meetings (same invitee email, canceled event followed by new event) and tag them as "rescheduled" to avoid double-counting.
- **UTM parameter availability:** UTM data is only captured if the Calendly booking page URL includes UTM parameters. Many bookings will not have UTM data — this is normal. The HubSpot cross-reference is the primary source attribution; UTMs are supplementary.
- **ICP scoring:** This module does not re-score leads for ICP fit. It relies on ICP scores already calculated by `directives/attribution.md` and `directives/lead_scoring.md`. If a meeting invitee's email matches a contact in those outputs, pull the existing ICP tier.
- **First run:** No previous snapshot exists. Save current state and skip week-over-week comparison.

---

## Scripts This Directive Feeds

- `execution/meeting_funnel_pull.py` — Steps 1, 2, 3, 4, 5, and 6
- `execution/analyze.py` — All reporting
