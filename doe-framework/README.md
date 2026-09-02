# DOE Framework - Marketing Intelligence Automation

The DOE (Directives → Orchestration → Execution) framework is a fully automated weekly marketing intelligence system for B2B SaaS companies. Every Sunday it pulls data from your ad platforms, CRM, search tools, Reddit, and LinkedIn, runs it through Claude for analysis, and delivers a structured PDF report to Slack. It replaces hours of manual weekly reporting with a single scheduled run that produces specific, data-backed recommendations - not dashboards you have to interpret yourself.

The framework is designed so that Claude Code acts as the orchestrator: it reads SOPs (directives), calls deterministic Python scripts (execution layer), handles errors, and assembles the final report. Each directive tells the execution scripts exactly what to pull, from where, and how to handle edge cases - so the system is self-documenting and maintainable without deep technical knowledge.

## What's Included

| Directive | What It Does |
|---|---|
| `keyword_intelligence.md` | Pulls Google Ads, GSC, and GA4 keyword data weekly. Scores every keyword against your ICP. Flags wasted spend on non-ICP terms and identifies page 2 recovery opportunities (positions 11-20 with high impressions). |
| `attribution.md` | Pulls all HubSpot deal and contact data. Maps every deal back to its original lead source (organic, paid, LinkedIn, Meta, direct, referral). Tracks pipeline velocity and stage movements. |
| `lead_scoring.md` | Pulls LinkedIn Ads and Meta Ads campaign performance and lead gen data. Cross-references every lead with HubSpot. Scores each lead against your ICP taxonomy (Tier 1/2/3/Unqualified). |
| `competitor_audit.md` | Scans SERP rankings for all your ICP keywords. Scrapes competitor ads on Meta and LinkedIn. Monitors competitor LinkedIn organic posts. Crawls top-ranking pages to identify content gaps. |
| `reddit_mining.md` | Mines Reddit for pain-point discussions in relevant subreddits. Tracks competitor mentions across all of Reddit. Surfaces real buyer language for ad copy and content strategy. |
| `weekly_report.md` | Takes all `.tmp/` data, sends it to Claude API in structured chunks, assembles a PDF report, and delivers it to Slack. This is the analysis and delivery layer. |
| `linkedin_organic.md` | Tracks your company page performance and thought leader posts via LinkedIn Marketing API and Apify. Benchmarks your organic engagement against competitors. |
| `conversion_funnel.md` | Maps your full GA4 visitor-to-conversion funnel. Detects zero-conversion landing pages with wasted paid spend. Flags ad copy vs. landing page mismatches. |
| `meeting_booking_funnel.md` | Pulls Calendly meeting data and cross-references with HubSpot. Tracks booking → completed → no-show rates by rep, event type, and lead source. |
| `email_sequence_performance.md` | Pulls HubSpot email sequence data. Reports per-step open and reply rates. Flags underperforming steps for rewrite. |
| `gbp_analysis.md` | Bi-weekly Google Business Profile audit. Analyzes categories, posts, reviews, photos, services, and descriptions for your company and competitors. Produces 8-week posting and photo calendars. |
| `entity_optimization.md` | Bi-monthly schema markup audit, Knowledge Graph presence check, and brand consistency review across all web touchpoints. |
| `langfuse_observability.md` | Wires Langfuse into every Claude API call in the analysis layer. Every weekly run appears as a trace in your Langfuse dashboard with full input/output, token counts, cost, and latency per call. |
| `langfuse_evals.md` | Builds a regression test suite on top of Langfuse Datasets. Catches output quality regressions when you change prompts, swap models, or update your ICP taxonomy - before bad output reaches your team. |

## How to Adopt

1. **Clone this repo** and copy the `doe-framework/` folder into your project
2. **Fill in `.env`** using `.env.example` as the template - add your API keys for every service you use
3. **Customize `CLAUDE.md`** - fill in your verticals, Slack channel, team member names, and delivery schedule
4. **Create `config/icp_taxonomy.py`** - define your ICP keywords organized by vertical (e.g., `{"Vertical 1": ["keyword a", "keyword b"], "Vertical 2": [...]}`) plus target job titles and minimum employee count for lead scoring
5. **Create `config/competitor_lists.py`** - add your actual competitors organized by vertical, each with `name`, `domain`, and `linkedin_url` fields. This drives SERP tagging, ad library scanning, and LinkedIn posts monitoring across all directives
6. **Edit each directive** in `directives/` - replace all `[YOUR_DOMAIN]`, `[YOUR_COMPANY]`, and account ID placeholders with your real values
7. **Build the execution scripts** - each directive ends with "Scripts This Directive Feeds" listing the Python scripts to create. Build one script at a time, testing each before moving on. The last one should be a main.py script that orchestrates the order of the workflow. 
8. **Run the orchestrator** - `python execution/main.py` - and review the output in `.tmp/`
9. **Schedule it** - deploy to Modal, a cron job, or GitHub Actions to run weekly

## Module Compatibility

Some directives are written for specific tools. Check the table below before you start so you know what applies to your stack.

| Module | Tool Requirement | If You Use Something Else |
|---|---|---|
| `attribution.md` | HubSpot | Tell Claude Code: *"I use [Salesforce/Pipedrive/etc.] - rewrite this directive for its API."* |
| `email_sequence_performance.md` | HubSpot Sequences | Same as above - or skip if you don't run outbound sequences |
| `meeting_booking_funnel.md` | Calendly + HubSpot | Tell Claude Code: *"I use [Chili Piper/Google Calendar/etc.] - rewrite this directive for its API."* |
| `weekly_report.md` (delivery) | Slack | Tell Claude Code to adapt the delivery step for Teams, email, or another channel |
| `gbp_analysis.md` | Optional: your own GBP listing | Competitor scraping still works without one - the "your company" audit sections will be skipped |
| `keyword_intelligence.md`, `lead_scoring.md`, `conversion_funnel.md` | Active Google Ads, Meta Ads, LinkedIn Ads accounts | Skip steps for platforms you don't run ads on - the other steps (GSC, GA4, organic) still produce output |
| All other modules | Platform-agnostic | No changes needed |

The framework's architecture works with any tool - the directives are just SOPs. If a directive is written for HubSpot and you use Salesforce, Claude Code can rewrite it: *"I use [tool] instead of [tool in directive]. Rewrite directives/[name].md for [tool]'s API."*

---

## Required Tools and APIs

| Tool | Used For | Link |
|---|---|---|
| Google Ads API | Keyword performance, campaign data | https://developers.google.com/google-ads/api |
| Google Search Console API | Organic rankings, impressions, CTR | https://developers.google.com/webmaster-tools |
| GA4 Data API | Website conversions, landing pages, funnel | https://developers.google.com/analytics/devguides/reporting/data |
| HubSpot Private App API | CRM deals, contacts, pipeline, sequences | https://developers.hubspot.com |
| LinkedIn Marketing API | Paid campaign performance, lead gen forms, company page metrics | https://learn.microsoft.com/en-us/linkedin/marketing |
| Meta Marketing API | Facebook/Instagram campaign performance and leads | https://developers.facebook.com/docs/marketing-apis |
| Calendly API | Meeting booking funnel data | https://developer.calendly.com |
| Apify | Web scraping for SERP, competitor ads, Reddit, LinkedIn posts, GBP | https://apify.com |
| Anthropic Claude API | Analysis layer - turns raw CSV data into actionable insights | https://docs.anthropic.com |
| Slack API | Delivering the weekly report PDF to a channel | https://api.slack.com |
| Langfuse | Observability and evals - traces every Claude call, tracks cost/latency, and runs regression tests on your analysis outputs | https://langfuse.com |

## Weekly Cost Estimate As of August 2026 (* Subject to change depending on Apify actor policies. Please note some Apify scrapers may not work too since they are constantly changing. Make sure to run tests before deploying.*)

| Component | Cost/Week |
|---|---|
| Apify (SERP scans, ad scraping, Reddit, LinkedIn posts, GBP) | ~$8 |
| Anthropic Claude API (12 analysis calls) | ~$5|
| Google Ads / GSC / GA4 / HubSpot / LinkedIn / Meta APIs | Free (within standard rate limits) |
| Calendly API | Free |
| **Total** | **~$13-17/week** |

GBP analysis adds ~$2/run bi-weekly. Entity optimization adds under $0.50 bi-monthly. Both are negligible.
