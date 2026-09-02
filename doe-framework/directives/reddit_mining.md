# Reddit Pain-Point Mining + Review Sentiment

## Goal
Every week, mine Reddit for pain-point discussions related to your product category and ICP verticals - surfacing real buyer language and unmet needs. This module captures the voice of the buyer and the voice of the market.

---

## When This Runs
- **Frequency:** Weekly (every Sunday as part of the main orchestrator)
- **Data window:** Last 7 days for Reddit posts/comments
- **Output:** Structured data saved to `.tmp/` for the analysis layer (`directives/weekly_report.md` → `execution/analyze.py`)

---

## Data Sources (Apify Actors)

**1. Reddit Scraper Lite**
- Actor ID: `trudax/reddit-scraper-lite`
- Used for: Pain-point discussions in relevant subreddits + competitor mentions across Reddit

**2. Competitor lists**
- Source: `config/competitor_lists.py`
- Used for: Tagging competitor mentions by sub-vertical

**3. ICP taxonomy**
- Source: `config/icp_taxonomy.py`
- Used for: Matching Reddit pain points to ICP buyer language and verticals

---

## Step 1: Reddit Subreddit Scraping

**What to do:**
Run the Reddit Scraper Lite (`trudax/reddit-scraper-lite`) on each target subreddit, searching for relevant keywords within each. Pull posts and comments from the last 7 days.

**Subreddit list:**
<!-- These subreddits are generic enough to be broadly applicable - adjust for your specific industry. -->
<!-- Add or remove subreddits based on where your buyers spend time. -->

Industry / problem-domain subreddits:
- reddit.com/r/fintech
- reddit.com/r/accounting
- reddit.com/r/automation
- reddit.com/r/artificialintelligence
- reddit.com/r/MachineLearning
- reddit.com/r/SaaS

Vertical-specific subreddits (add based on your ICP verticals):
<!-- Examples: -->
<!-- - reddit.com/r/mortgage -->
<!-- - reddit.com/r/insurance -->
<!-- - reddit.com/r/logistics -->
<!-- - reddit.com/r/supplychain -->
- reddit.com/r/[your vertical 1]
- reddit.com/r/[your vertical 2]

**Search keywords within each subreddit:**
<!-- Replace with keywords related to your product category and buyer pain points -->
<!-- Examples: -->
<!-- - document processing -->
<!-- - OCR automation -->
<!-- - invoice automation -->
<!-- - manual data entry -->
- [keyword 1 - your product category]
- [keyword 2 - buyer pain point]
- [keyword 3 - industry pain point]
- [keyword 4 - competitor category name]

**For each result, record:**
- Subreddit name
- Post title
- Post body text
- Post author
- Post date
- Post URL
- Upvotes
- Number of comments
- Top comments (up to 10 per post, with comment text, author, upvotes)

**Save to:** `.tmp/reddit_subreddit_posts.csv`

---

## Step 2: Reddit Competitor Mention Searches

**What to do:**
Run Reddit-wide searches for competitor names to find discussions where people mention, recommend, complain about, or compare competitors.

**Search list (one per competitor + your own company):**
- reddit.com/search/?q=[COMPETITOR_1]&sort=new
- reddit.com/search/?q=[COMPETITOR_2]&sort=new
- reddit.com/search/?q=[COMPETITOR_3]&sort=new
- reddit.com/search/?q=[COMPETITOR_4]&sort=new
- reddit.com/search/?q=[COMPETITOR_5]&sort=new
- reddit.com/search/?q=[YOUR_COMPANY]&sort=new

**For each result, record:**
- Search term (competitor name)
- Subreddit where the mention appeared
- Post title
- Post body text (or relevant comment text)
- Post date
- Post URL
- Upvotes
- Sentiment: positive, negative, neutral, or comparison (the analysis layer will classify this - the scraper just pulls the raw text)

**Compare against previous week's snapshot (stored in `.tmp/reddit_mentions_snapshot.json`):**
- Flag any competitor with a spike in mentions (more than 2x their average weekly mentions)
- Flag any negative sentiment surge about a competitor (potential opportunity)
- Flag any mention of your own company (always surface these regardless of volume)
- On first run, save snapshot and skip change detection

**Save to:** `.tmp/reddit_competitor_mentions.csv` and `.tmp/reddit_mentions_snapshot.json`

---

## Step 3: Cross-Reference with ICP Taxonomy and Competitor Lists

**Reddit pain-point tagging:**
Using `config/icp_taxonomy.py`, tag each Reddit post/comment with the most relevant vertical based on:
- Which subreddit it appeared in (e.g., r/[your vertical subreddit] → [vertical name])
- Which keywords matched (e.g., "keyword from your ICP taxonomy" → matching vertical)
- Whether the pain point matches known buyer language from the ICP taxonomy

**Save tags as additional columns in:** `.tmp/reddit_subreddit_posts.csv`, `.tmp/reddit_competitor_mentions.csv`

---

## Output Files

| File | Contents |
|---|---|
| `.tmp/reddit_subreddit_posts.csv` | All Reddit posts/comments from subreddit keyword searches with vertical tags |
| `.tmp/reddit_competitor_mentions.csv` | All Reddit competitor mentions with sentiment indicators |
| `.tmp/reddit_mentions_snapshot.json` | Current competitor mention counts for next week's comparison |

---

## What the Analysis Layer Does With This Data

**For Section 1.4 and 2.4 of the weekly report (per vertical):**
- Top pain points from Reddit this week: the 5 most upvoted or discussed pain points, grouped by vertical
- Buyer language tracker: exact phrases and terminology real buyers are using (feeds into ad copy and content strategy)
- Competitor mention summary: which competitors are being discussed, in what context (recommendation, complaint, comparison)
- Competitor sentiment shifts: any competitor seeing a spike in negative or positive discussion

**For Section 3 (Cross-Vertical Summary):**
- Emerging pain points: any new pain point appearing across multiple subreddits or verticals that your company doesn't currently address in its messaging
- Competitor vulnerability alerts: competitors receiving Reddit complaints that you could capitalize on

---

## Edge Cases and Notes

- **Apify costs:** Reddit Scraper Lite for subreddit/keyword combinations costs approximately $0.50/week for a typical setup.
- **Reddit Scraper rate limiting:** The Reddit Scraper Lite uses custom proxy configuration for cheaper scraping. If blocked or rate-limited, retry once after 60 seconds. If it fails again, log the error and continue with collected data.
- **Sentiment classification:** The scraper pulls raw text only. All sentiment classification (positive, negative, neutral, comparison) is done by the analysis layer (`execution/analyze.py` using Claude API). The scraper does not attempt sentiment analysis.
- **Reddit data freshness:** Reddit posts can appear and get engagement over several days. The 7-day window captures most relevant discussion.
- **First run:** No previous snapshots exist. Save current state and skip all change detection.

---

## Scripts This Directive Feeds

- `execution/reddit_scrape.py` - Steps 1 and 2
- `execution/analyze.py` - Step 3 (tagging) and all analysis layer reporting
