# Build Guide: Phase 1 Through 9

This guide walks you through building the full automation from scratch. No prior coding experience required - Claude Code writes all the Python. Your job is to configure, review, and test.

| Phase | What | Time | Requires Code? |
|---|---|---|---|
| 1 | Folder + GitHub setup | 10 min | No - terminal commands only |
| 2 | Build ICP config files | 1 hr | Claude Code does it |
| 3 | Write all 12 directives | 1 day | No - written in chat |
| 4 | Get all API keys | 2–7 days | No - web forms |
| 5 | Build execution scripts | 2-3 days | Claude Code does it |
| 6 | Test each module | 2 days | Claude Code does it |
| 7 | Connect everything (main.py) | 1 day | Claude Code does it |
| 8 | Deploy to Modal + first test | 1 hour | Claude Code does it |
| 9 | Go live | Done | Runs automatically |

> **Start Phase 4 on Day 1 in parallel with Phase 3.** Google Ads Developer Token and LinkedIn Marketing API both require approval that can take 3–7 days. Don't wait until Phase 3 is done to start them.

---

## Phase 1 - Folder + GitHub Setup

**Before you start:**
1. Install [Claude Code](https://claude.ai/code) if you haven't already. Everything in this guide is built inside Claude Code.
2. Create a free [GitHub account](https://github.com/signup) if you don't have one.
3. Install the [GitHub CLI](https://cli.github.com/) and authenticate - this is the easiest way to set up GitHub access from the terminal:
```bash
gh auth login
```
Follow the prompts - it opens a browser and handles everything. You only need to do this once.

4. Create a new empty repo at [github.com/new](https://github.com/new). Keep it private until you're ready to share.

Create your project folder and connect it to GitHub so your work is version-controlled from the start.

```bash
mkdir [your-project-name]
cd [your-project-name]
git init
git remote add origin https://github.com/[YOUR_GITHUB_USERNAME]/[your-project-name].git
```

Create a `.gitignore` to protect secrets:
```
.env
.tmp/
__pycache__/
*.pyc
service_account.json
```

Copy the `doe-framework/CLAUDE.md` template into your project root and fill in the placeholders (your verticals, Slack channel, team members, and delivery schedule). This is how Claude Code knows your project context.

Create your `.env` file using `.env.example` as the template. Never commit this file.

Create a `requirements.txt` - leave it empty for now. Claude Code will add packages to it as it builds each script. Before Phase 6 testing, run:
```bash
pip install -r requirements.txt
```

Create the folder structure:
```
[your-project-name]/
├── CLAUDE.md
├── .env
├── .gitignore
├── requirements.txt
├── directives/
├── execution/
├── config/
└── .tmp/
```

---

## Saving Your Work to GitHub

Commit and push at the end of each phase. Your `.env` file is already in `.gitignore` - it will never be committed.

```bash
git add .
git commit -m "Phase [N] complete"
git push origin main
```

Do this after Phase 2 (config files), after Phase 3 (directives), after each script you finish in Phase 5, and after Phase 7 (main.py). The goal is that if you ever need to start over or hand the project off, everything is recoverable from GitHub except your `.env` keys.

---

## Useful Claude Code Commands

These slash commands are available inside Claude Code at any time. Type them directly in the chat input.

| Command | When to Use |
|---|---|
| `/clear` | Start a fresh conversation - use at the start of each new phase so old context doesn't bleed in |
| `/compact` | You're mid-phase and context is getting heavy but you're not done yet - compacts the conversation without losing your current work |
| `/model` | Switch models mid-session - use `opus` for complex debugging, `sonnet` for standard script building |
| `/usage` | Check how many tokens you've used and estimated cost for the current session |
| `/mcp` | See which MCP tool connections are active and whether they're working |
| `/init` | Generate a CLAUDE.md for your project - useful if you skipped Phase 1 setup |
| `/review` | Ask Claude to review the code it just wrote before you test it |
| `/security-review` | Run a security check on your scripts - useful after Phase 5 to make sure no keys are hardcoded |

**Workflow tip for long builds:** At the start of each phase, use `/clear` to start fresh. If you hit the context limit mid-phase, use `/compact` to compress - it keeps your current work in memory. Only start a new session when you've fully finished a phase and everything is committed to GitHub.

---

## Phase 2 - Build ICP Config Files

Before writing directives, you need two config files that the rest of the system depends on:

**`config/icp_taxonomy.py`** - Your ICP keywords, target job titles, and company size thresholds. This is the scoring engine. Every keyword, lead, and campaign gets scored against this file.

**`config/competitor_lists.py`** - Your competitors organized by sub-vertical. This drives SERP tagging, ad library scanning, and LinkedIn monitoring.

**How to build them:**
Open Claude Code and say: *"Help me build config/icp_taxonomy.py. I want to define my ICP verticals, target keywords per vertical, target job titles, and minimum company size for Tier 1-3 leads."*

Then: *"Now help me build config/competitor_lists.py with my competitors organized by vertical, each with name, domain, and LinkedIn URL."*

These files come from your own knowledge of your market - Claude Code structures them, you fill in the content. Reference your CRM closed-won deal data and any sales call recordings you have to ground these in reality rather than guesswork.

---

## Phase 3 - Write All 12 Directives

The directive templates in `directives/` are your starting point - they're already structured. Your job is to replace every placeholder with your real values. Do them one at a time in Claude Code. Review each one before saving. The directives are your SOPs - the better they are, the better the output.

**How to prompt Claude Code for each directive:**
> *"Open directives/[directive_name].md. Replace all placeholders - [YOUR_COMPANY], [YOUR_DOMAIN], [COMPETITOR_1], [YOUR_VERTICAL_1], etc. - with my real values. Here's what to use: [paste your company name, domain, competitor names, verticals]. Save the file when done."*

| Order | Directive File | Notes |
|---|---|---|
| 1 | `keyword_intelligence.md` | Most complex - start here |
| 2 | `attribution.md` | CRM closed-loop attribution |
| 3 | `lead_scoring.md` | Ad platform lead scoring |
| 4 | `competitor_audit.md` | SERP + ad libraries + content gap analysis |
| 5 | `reddit_mining.md` | Reddit pain points + competitor mentions |
| 6 | `gbp_analysis.md` | Most detailed - 7 sub-modules |
| 7 | `conversion_funnel.md` | GA4 + landing page content analysis |
| 8 | `linkedin_organic.md` | LinkedIn page + thought leader performance |
| 9 | `meeting_booking_funnel.md` | Calendly data |
| 10 | `email_sequence_performance.md` | CRM email sequences |
| 11 | `weekly_report.md` | Report generation + Slack delivery |
| 12 | `entity_optimization.md` | Schema + knowledge graph (bi-monthly) |

Use the directive templates in `directives/` as your starting point. Replace every `[YOUR_COMPANY]`, `[YOUR_DOMAIN]`, `[COMPETITOR_1]` etc. placeholder with your real values.

---

## Phase 4 - Get All API Keys

Get these in parallel with Phase 3. Add each key to your `.env` file as you receive it. You can ask Claude in Claude chat how to get create and get these keys.

| # | Credential | Difficulty | Time | Notes |
|---|---|---|---|---|
| 1 | Google Cloud Project + Service Account | Easy | 20 min | Covers GSC, GA4, Google Ads |
| 2 | Google Search Console Access | Easy | 10 min | Add service account email as user - it's the `client_email` field in your downloaded `service-account.json` |
| 3 | GA4 Access | Easy | 10 min | Add service account email as viewer - same `client_email` value |
| 4 | Google Ads Developer Token | Medium | **1–3 day wait** | **START THIS ON DAY 1** |
| 5 | Meta Marketing API | Medium | 30 min | System user + token |
| 6 | LinkedIn Marketing API | Medium | **3–7 day wait** | **START THIS ON DAY 1** |
| 7 | Apify Account + API Token | Easy | 20 min | Start on Free plan, upgrade to Starter before Phase 6 |
| 8 | Anthropic API Key | Easy | 10 min | Usage-based billing |
| 9 | CRM Private App (HubSpot or equivalent) | Easy | 15 min | Read access to CRM objects |
| 10 | Scheduling Tool API (Calendly or equivalent) | Easy | 10 min | Generate in integrations |
| 11 | Slack Webhook + Bot Token | Easy | 10 min | Incoming webhook to your delivery channel |

> Start credentials 4 and 6 on Day 1 - they require manual review and can block your Phase 5 start if you wait.

### Apify Actor Setup

You need 8 Apify actors. You do not download anything - just click the actor link, click "Try for free," and it's added to your account. Claude Code calls them via API in Phase 5.

| Actor | Actor ID | Used By |
|---|---|---|
| Google Search Scraper | `apify/google-search-scraper` | competitor_audit.md, entity_optimization.md |
| Google Maps Scraper | `compass/crawler-google-places` | gbp_analysis.md |
| Reddit Scraper Lite | `trudax/reddit-scraper-lite` | reddit_mining.md |
| Facebook Ad Library Scraper | `apify/facebook-ads-scraper` | competitor_audit.md |
| Website Content Crawler | `apify/website-content-crawler` | competitor_audit.md, conversion_funnel.md, entity_optimization.md |
| LinkedIn Company Posts Scraper | Search Apify Store | competitor_audit.md, linkedin_organic.md |
| LinkedIn Profile Posts Scraper | Search Apify Store | linkedin_organic.md |
| LinkedIn Ads Scraper | Search Apify Store | competitor_audit.md |

> **Note on LinkedIn actors:** LinkedIn scrapers on Apify break and get replaced frequently due to LinkedIn's anti-scraping policies. Do not rely on a fixed actor ID - search the Apify Store for the current best-rated option for each use case before you start Phase 5. Once you've selected actors, update the actor IDs in `directives/competitor_audit.md` and `directives/linkedin_organic.md` before asking Claude Code to build those scripts.

---

## Phase 5 - Build Execution Scripts

Build all Python scripts using Claude Code. One module at a time - never try to build everything at once. After building each script, go to Phase 6 to test it before starting the next one.

**How to prompt Claude Code for each script:**
> *"Read directives/[directive_name].md. Build execution/[script_name].py that implements Steps 1 through N. Save output to .tmp/ as specified in the directive. Do not build the analysis layer yet - just the data pull."*

| Order | Script(s) | Directive to Read First |
|---|---|---|
| 1st | `execution/hubspot_pull.py` | `directives/attribution.md` |
| 2nd | `execution/gsc_pull.py` | `directives/keyword_intelligence.md` |
| 3rd | `execution/google_ads_pull.py` | `directives/keyword_intelligence.md` |
| 4th | `execution/meta_pull.py` | `directives/lead_scoring.md` |
| 5th | `execution/linkedin_pull.py` | `directives/lead_scoring.md` |
| 6th | `execution/ga4_pull.py` + `conversion_funnel_pull.py` | `directives/conversion_funnel.md` |
| 7th | `execution/competitor_serp_scan.py` | `directives/competitor_audit.md` |
| 8th | `execution/reddit_scrape.py` | `directives/reddit_mining.md` |
| 9th | `execution/gbp_scrape.py` | `directives/gbp_analysis.md` |
| 10th | `execution/meeting_funnel_pull.py` | `directives/meeting_booking_funnel.md` |
| 11th | `execution/email_sequence_pull.py` | `directives/email_sequence_performance.md` |
| 12th | `execution/linkedin_organic_pull.py` | `directives/linkedin_organic.md` |
| 13th | `execution/entity_check.py` | `directives/entity_optimization.md` |
| 14th | `execution/analyze.py` | `directives/weekly_report.md` - Claude API analysis layer |
| 14a | `execution/langfuse_client.py` | `directives/langfuse_observability.md` - build after `analyze.py`, then go back and wire it in |
| 14b | `execution/evals/` + `scripts/setup_langfuse_datasets.py` + `.github/workflows/doe_evals.yml` | `directives/langfuse_evals.md` - build after `langfuse_client.py` is working |
| 15th | `execution/generate_report.py` | `directives/weekly_report.md` - PDF generation |
| 16th | `execution/slack_delivery.py` | `directives/weekly_report.md` - Slack delivery |
| 17th | `execution/main.py` | Master orchestrator - build last |

---

## Phase 6 - Test Each Module

After building each script in Phase 5, test it immediately before moving on. A script passes when it produces a `.csv` or `.json` file in `.tmp/` with real data from your actual accounts.

**How to prompt Claude Code to test:**
> *"Run a production-grade test of execution/[script_name].py. Pull real data from my accounts, show me the first 5 rows of output, flag any errors, rate limit issues, or missing fields, and confirm the output file was saved to .tmp/."*

**Self-annealing loop - when something breaks:**
1. Fix the script
2. Test it again
3. Update the directive if the fix revealed something the directive got wrong
4. Move on

Do not move to Phase 7 until every script produces real output without errors or rate limit failures.

**Before starting Phase 7:** Seed your Langfuse eval datasets. After your first successful test run of `analyze.py`, identify one output from each of the four key calls (keyword intelligence, lead scoring, competitor audit, cross-vertical summary) that you've reviewed and confirmed is correct. Use `promote_to_dataset` to save each one as a golden example. You only need one item per dataset to start - you can add more over time. Once seeded, run:
```bash
python execution/evals/run_evals.py
```
All four should pass. This is your baseline. From this point forward, any prompt change you make will be automatically checked against these golden examples in CI.

---

## Phase 7 - Connect Everything

In Claude Code say:
> *"Read directives/weekly_report.md. Update execution/main.py to run all scripts in the correct order. Add error handling so if one script fails, it logs the error and continues with the remaining scripts. The orchestrator should produce a run summary at the end showing which modules succeeded and which failed."*

Test the full pipeline with:
```bash
python execution/main.py
```

Verify the PDF is generated and delivered to Slack.

---

## Phase 8 - Deploy to Modal

Modal runs your automation on a schedule in the cloud for ~$2–5/month.

**Step 1:** Install Modal and create an account:
```bash
pip install modal
modal setup
```
This opens a browser to create your free Modal account and links it to your terminal. You only need to do this once.

**Step 2:** Add your `.env` keys to Modal Secrets:
```bash
modal secret create [your-project]-secrets \
  ANTHROPIC_API_KEY=... \
  APIFY_API_TOKEN=... \
  # (all other keys)
```

**Step 3:** Tell Claude Code to add the Modal deployment wrapper:
> *"Add a Modal deployment wrapper to execution/main.py. Schedule it to run every Sunday at 8:30 PM US Pacific. Use the secret named [your-project]-secrets."*

**Step 4:** Deploy:
```bash
modal deploy execution/main.py
```

**Step 5:** Test manually:
```bash
modal run execution/main.py
```

Verify the report arrives in your Slack channel.

---

## Phase 9 - Go Live

The automation runs on schedule. No action required.

Monitor the first few runs to verify output quality. After 2–3 weeks you'll have enough data for week-over-week comparisons to start working. That's when the report becomes genuinely actionable.

**Known gaps to add in a Phase 2 expansion:**
- Backlink data (requires Ahrefs or SEMrush API, ~$99+/month) - explains WHY competitors outrank you, not just that they do
- Technical SEO health (Core Web Vitals, crawl errors, mobile usability) - the GSC pull can be expanded to cover some of this without new tools
