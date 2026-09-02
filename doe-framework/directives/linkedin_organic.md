# LinkedIn Organic Performance

## Goal
Every week, track your company's LinkedIn organic performance - company page posts and thought leader posts. Measure engagement rates, follower growth, and content performance. Benchmark against competitor organic activity (data pulled by `directives/competitor_audit.md`). Identify what content themes and formats drive the most engagement for your company specifically.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days for new posts, plus rolling engagement data on recent posts
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources

**1. LinkedIn Marketing API (your own company page data)**
- Account ID: `[YOUR_LINKEDIN_ACCOUNT_ID]`
- Company Page URL: `https://www.linkedin.com/company/[your-company-slug]/`
- Used for: Your own post performance, follower metrics, and page analytics

**2. LinkedIn Profile Posts Scraper (Apify - for thought leader personal posts)**
- Actor ID: `[YOUR_LINKEDIN_PROFILE_POSTS_ACTOR_ID]` - search Apify Store for "LinkedIn profile posts scraper" and use the current best-rated option. As of May 2026: `harvestapi/linkedin-profile-posts`
- Used for: Scraping individual thought leader personal profile posts that the LinkedIn Marketing API does not cover

**3. LinkedIn Company Posts Scraper (Apify - backup for company page data)**
- Actor ID: `[YOUR_LINKEDIN_POSTS_ACTOR_ID]` - same actor selected for `directives/competitor_audit.md`. As of May 2026: `harvestapi/linkedin-company-posts`
- Used for: Backup source for company page posts if the LinkedIn API fails, and for loading competitor comparison data produced by `directives/competitor_audit.md`

**4. Competitor organic post data**
- Source: `.tmp/competitor_linkedin_posts.csv` (produced by `directives/competitor_audit.md` → Step 4)
- Used for: Benchmarking your organic performance against competitors

---

## Step 1: Company Page Performance (LinkedIn API)

**What to pull from the LinkedIn Marketing API for your company page:**

**Page-level metrics (last 7 days):**
- Total followers (current count)
- New followers gained this week
- Page views
- Unique visitors
- Custom button clicks (if configured)

**Post-level metrics (for all posts published in the last 30 days):**
- Post date
- Post content (full text)
- Post type (text, image, video, document/carousel, article, poll)
- Impressions
- Unique impressions (if available)
- Clicks
- Reactions (total, and broken out by type if available: like, celebrate, support, insightful, etc.)
- Comments
- Shares
- Engagement rate (calculated: (reactions + comments + shares + clicks) ÷ impressions)
- Video views and completion rate (if video post)

**Why 30 days instead of 7:** Posts published 2-3 weeks ago may still be accumulating engagement. Pulling 30 days of posts captures the full engagement curve for recent content.

**Save to:** `.tmp/company_linkedin_page_metrics.csv` (page-level) and `.tmp/company_linkedin_posts.csv` (post-level)

---

## Step 2: Thought Leader Post Tracking

**Why this matters:** For many B2B companies, individual employees posting as thought leaders generate more engagement than company page posts. The LinkedIn Marketing API does not provide data on personal profiles - only the company page.

**Thought leaders to track:**
<!-- Replace with your own thought leaders' names and LinkedIn profile URLs -->
<!-- Remove this section entirely if you don't track individual thought leaders -->

| Name | LinkedIn Profile URL |
|---|---|
| [THOUGHT_LEADER_1] | `https://www.linkedin.com/in/[profile-slug-1]/` |
| [THOUGHT_LEADER_2] | `https://www.linkedin.com/in/[profile-slug-2]/` |
| [THOUGHT_LEADER_3] | `https://www.linkedin.com/in/[profile-slug-3]/` |

**What to do:**
Run the LinkedIn Profile Posts Scraper (`[YOUR_LINKEDIN_PROFILE_POSTS_ACTOR_ID]`) for each thought leader profile URL. Pull posts from the last 30 days.

**For each post, record:**
- Author name
- Author profile URL
- Post date
- Post content (full text)
- Post type (text, image, video, document/carousel, article)
- Reactions count
- Comments count
- Shares count (if available)
- Post URL
- Whether the post mentions your company, your product category, or ICP-related topics (tagged by the analysis layer, not the scraper)

**Save to:** `.tmp/thought_leader_posts.csv`

---

## Step 3: Data Sharing with Competitor Audit

**What to do:**
Load competitor organic post data from `.tmp/competitor_linkedin_posts.csv` (produced by `directives/competitor_audit.md` → Step 4).

If the competitor audit module has already run this week (which it should - the orchestrator runs it first), the data is already available. If not, log a warning and skip the competitive benchmarking portion of the analysis.

Additionally, if your company's posts were already scraped in `directives/competitor_audit.md` Step 4, that data can supplement the LinkedIn API data from Step 1. The API data is more authoritative (it includes impressions, clicks, and engagement rate), but the scraped data serves as a backup if the API fails.

**No additional scraping needed in this step - just data loading.**

---

## Step 4: Week-Over-Week Comparison

**What to do:**
Compare this week's metrics against last week's snapshot (stored in `.tmp/linkedin_organic_snapshot.json`).

**Flag and report:**
- **Follower growth:** Net new followers this week vs. last week. Is growth accelerating or slowing?
- **Posting frequency change:** Did the company post more or fewer times this week vs. last week?
- **Engagement rate change:** Average engagement rate this week vs. last week (across all company page posts)
- **Top performing post:** The post with the highest engagement rate this week (across company page and thought leader posts combined)
- **Lowest performing post:** The post with the lowest engagement rate this week (if more than 2 posts were published)
- **Thought leader activity:** Which thought leaders posted this week? Any thought leader who did not post? How many posts per thought leader?
- **Thought leader engagement trend:** Each thought leader's average engagement this week vs. last week

**Save current state as:** `.tmp/linkedin_organic_snapshot.json`
**Save changes as:** `.tmp/linkedin_organic_changes.json`

On first run, save snapshot and skip change detection.

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/company_linkedin_page_metrics.csv` | Company page follower and visitor metrics |
| `.tmp/company_linkedin_posts.csv` | All company page posts (last 30 days) with full engagement data |
| `.tmp/thought_leader_posts.csv` | All thought leader personal posts (last 30 days) with engagement data |
| `.tmp/linkedin_organic_snapshot.json` | Current organic state for next week's comparison |
| `.tmp/linkedin_organic_changes.json` | Week-over-week changes in followers, posting frequency, engagement |

---

## What the Analysis Layer Does With This Data

**For Section 1.8 and 2.8 of the weekly report:**
- **Page health:** Follower count, new followers this week, page views, visitor trend
- **Post performance summary:** Total posts this week (company page + thought leaders), average engagement rate, best and worst performing posts with specific content and metrics
- **Content format analysis:** Which post types (text, image, video, carousel) generate the highest engagement - based on rolling 30-day data, not just this week
- **Thought leader leaderboard:** Ranked by engagement rate this week. Show each thought leader's post count, average reactions, top post. Flag any thought leader who hasn't posted.
- **Competitive benchmark:** Your company's average engagement rate vs. competitor average engagement rate. Your posting frequency vs. competitor average.
- **Content theme analysis:** Which topics drive the most engagement? Uses ICP taxonomy keywords from `config/icp_taxonomy.py` to classify post topics.
- **Recommendations:** Specific suggestions for next week's content - what topics, formats, and posting cadence to prioritize based on what's working

**For Section 3 (Cross-Vertical Summary):**
- LinkedIn organic health: quick summary of engagement trend and follower growth
- Any competitor significantly outperforming your company on organic LinkedIn

---

## Edge Cases and Notes

- **LinkedIn API rate limits:** The LinkedIn Marketing API allows 100 requests per day per user. Page metrics and post metrics for a single company page will use a small number of requests. Not a concern for weekly pulls.
- **LinkedIn API scope:** The LinkedIn Marketing API returns data for your company page only - not personal profiles. This is why thought leader tracking requires a separate Apify LinkedIn profile posts actor (`[YOUR_LINKEDIN_PROFILE_POSTS_ACTOR_ID]`).
- **Two different Apify actors:** This module uses two separate LinkedIn scrapers: `[YOUR_LINKEDIN_PROFILE_POSTS_ACTOR_ID]` for thought leader personal profiles, and `[YOUR_LINKEDIN_POSTS_ACTOR_ID]` for company page posts and competitor data. They are different actors with different inputs - do not mix them up.
- **Thought leader profile URLs are hardcoded:** Unlike competitor lists which are dynamic, the thought leader profile URLs are fixed in this directive. If a thought leader leaves or a new one is added, update this directive manually.
- **Engagement rate calculation:** Use (reactions + comments + shares + clicks) ÷ impressions for LinkedIn API data. For scraped data (thought leader posts and competitor posts), clicks and impressions are typically not available - use (reactions + comments + shares) as the engagement metric instead. Note which formula was used so the analysis layer doesn't compare API-sourced engagement rates directly against scraper-sourced engagement counts.
- **Post attribution to verticals:** The analysis layer will classify posts by topic using ICP keywords from `config/icp_taxonomy.py`. A post mentioning a vertical-specific keyword is tagged as that vertical's content. Posts about company culture, hiring, or events are tagged as "brand."
- **Competitor data dependency:** Competitive benchmarking requires `.tmp/competitor_linkedin_posts.csv` from `directives/competitor_audit.md`. The orchestrator (`execution/main.py`) must ensure the competitor audit module runs before this module. If competitor data is not available, skip the benchmarking section and note it.
- **Apify costs:** A small number of thought leader profiles × weekly scrape = minimal cost (under $0.50/week). No additional cost for competitor data since it's already pulled by `directives/competitor_audit.md`.
- **30-day post window:** Pulling 30 days of posts each week means the same posts appear in multiple weekly pulls. The script should update engagement metrics for existing posts (reactions may have increased since last week) rather than creating duplicate rows. Use post URL as the unique key.
- **First run:** No previous snapshot exists. Save current state and skip week-over-week comparison.

---

## Scripts This Directive Feeds

- `execution/linkedin_organic_pull.py` - Steps 1, 2, and 4
- `execution/analyze.py` - Step 3 (data loading and benchmarking) and all reporting
