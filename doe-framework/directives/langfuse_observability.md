# Langfuse Observability

## Goal
Every Claude API call made by `execution/analyze.py` should be traced in Langfuse. Each weekly run should appear as a single trace containing all 12–13 analysis calls as nested generations. Every generation captures the input prompt, output text, token counts, cost, and latency automatically. This gives you a permanent, searchable record of every analysis the system has ever produced, with cost and performance data over time.

---

## When This Runs
- **Always** - this is not a module that runs on its own schedule. It is a shared client that wraps every Claude API call in `execution/analyze.py`.
- No data is written to `.tmp/`. All output goes directly to your Langfuse project dashboard.

---

## Prerequisites

**1. Langfuse account**
- Sign up at [cloud.langfuse.com](https://cloud.langfuse.com) (EU region) or [us.cloud.langfuse.com](https://us.cloud.langfuse.com) (US region)
- Go to **Project Settings → API Keys** and generate a key pair
- Copy `Secret Key` and `Public Key` into your `.env` file

**2. Environment variables**
The following must be present in `.env` before building this script:
```
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_HOST=https://us.cloud.langfuse.com
```
Use `https://cloud.langfuse.com` if you are on the EU region.

**3. Python package**
```bash
pip install langfuse
```
Add `langfuse>=2.0.0` to `requirements.txt`.

---

## Step 1: Build the Shared Langfuse Client

Build `execution/langfuse_client.py`. This file is imported by `analyze.py` - it is not run directly.

**What this file must contain:**

**1. A Langfuse instance** initialized from environment variables:
```python
from langfuse import Langfuse
import os

langfuse = Langfuse(
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "https://us.cloud.langfuse.com"),
)
```

**2. A `call_claude` wrapper function** with this exact signature:
```python
def call_claude(
    *,
    client,           # the Anthropic client instance
    trace,            # the active Langfuse trace for this run
    call_name: str,   # e.g. "call_1_keyword_intelligence"
    model: str,
    system: str,
    messages: list[dict],
    max_tokens: int = 8192,
    temperature: float | None = 0,
) -> str:
```

**What `call_claude` must do:**
1. Open a Langfuse generation span using `trace.generation(name=call_name, model=model, model_parameters={...}, input={...})`
2. Call `client.messages.create(...)` with all parameters
3. Extract the response text from `response.content[0].text`
4. Call `generation.end(output=text, usage={"input": response.usage.input_tokens, "output": response.usage.output_tokens, "unit": "TOKENS"})`
5. Return the response text

The function should let all Anthropic API exceptions propagate naturally - do not swallow errors here. Error handling lives in `analyze.py`.

**3. A `flush` function** that calls `langfuse.flush()`:
```python
def flush():
    langfuse.flush()
```
This is called at the end of every `analyze.py` run to ensure all events are sent before the process exits.

---

## Step 2: Update `analyze.py` to Use the Shared Client

In `execution/analyze.py`, make the following changes:

**At the top of the file:**
```python
from langfuse_client import langfuse, call_claude, flush
import datetime
```

**At the start of the analysis run (before any Claude calls):**
```python
run_date = datetime.date.today().isoformat()  # e.g. "2026-08-31"

trace = langfuse.trace(
    name="doe_weekly_run",
    session_id=f"weekly-{run_date}",
    metadata={"run_date": run_date},
    tags=["weekly", "doe-framework"],
)
```

**Replace every `client.messages.create(...)` call** with `call_claude(...)`:
```python
# Before:
response = client.messages.create(
    model=MODEL_DEFAULT,
    system=SYSTEM_PROMPT,
    messages=[{"role": "user", "content": keyword_prompt}],
    max_tokens=8192,
    temperature=0,
)
text = response.content[0].text

# After:
text = call_claude(
    client=client,
    trace=trace,
    call_name="call_1_keyword_intelligence",
    model=MODEL_DEFAULT,
    system=SYSTEM_PROMPT,
    messages=[{"role": "user", "content": keyword_prompt}],
    max_tokens=8192,
    temperature=0,
)
```

Use these `call_name` values for each call:

| Claude Call | `call_name` value |
|---|---|
| Call 1: Keyword Intelligence | `call_1_keyword_intelligence` |
| Call 2: Lead Scoring | `call_2_lead_scoring` |
| Call 3: Competitor Audit | `call_3_competitor_audit` |
| Call 4: Reddit + Sentiment | `call_4_reddit_sentiment` |
| Call 5: GBP Analysis | `call_5_gbp_analysis` |
| Call 6: Attribution | `call_6_attribution` |
| Call 7: Conversion Funnel | `call_7_conversion_funnel` |
| Call 8: LinkedIn Organic | `call_8_linkedin_organic` |
| Call 9: Meeting Booking Funnel | `call_9_meeting_funnel` |
| Call 10: Email Sequence | `call_10_email_sequence` |
| Call 11: Cross-Vertical Summary | `call_11_cross_vertical` |
| Call 12: Entity Health | `call_12_entity_health` |

**At the end of `analyze.py`** (in a `finally` block so it always runs):
```python
finally:
    flush()
```

---

## What You Will See in Langfuse

**Traces view (`/traces`):**
- One row per weekly run, named `doe_weekly_run`
- Click any row → see all 12–13 Claude calls nested inside it as generations
- Each generation shows: input prompt, full output text, input tokens, output tokens, cost (auto-calculated by Langfuse), latency in ms

**Dashboard (`/dashboard`):**
- Cost over time - total Claude API spend per week
- Token usage by model - Sonnet vs Opus breakdown
- Latency percentiles - P50 and P95 per call

**Sessions (`/sessions`):**
- Each run is grouped under `session_id = "weekly-2026-08-31"` etc.
- Compare two different weeks side by side

---

## Edge Cases and Notes

- **If Langfuse is unreachable:** The `call_claude` wrapper should still return the Claude response. A Langfuse connection failure should never crash the weekly run. Wrap the `trace.generation()` and `generation.end()` calls in a try/except that logs a warning but does not raise.
- **Cost calculation:** Langfuse auto-calculates cost from token counts if the model is in its price list. If you are using a new Claude model that is not yet in Langfuse's list, add a model definition manually at `cloud.langfuse.com → Settings → Models`.
- **Temperature=None for Call 11:** The cross-vertical summary call omits `temperature` to let the model use its default. Pass `temperature=None` to `call_claude` and have the wrapper omit the parameter from the Anthropic call when it is `None`.
- **First run:** No historical data to compare against. This is expected. The dashboard will be sparse until you have 2–3 runs.

---

## Output

No files are written to `.tmp/`. All output is sent to your Langfuse project at the host configured in `.env`.

To verify the integration is working after the first run: log into Langfuse, go to **Traces**, and confirm you see a `doe_weekly_run` trace with 12–13 nested generations.

---

## Scripts This Directive Feeds

- `execution/langfuse_client.py` - **[NEW]** Shared Langfuse client and `call_claude` wrapper
- `execution/analyze.py` - **[MODIFY]** Import `langfuse_client`, create trace at run start, replace all `client.messages.create` calls with `call_claude`, flush at end
