# Langfuse Evals + Regression Testing

## Goal
Build a regression test suite that catches output quality problems before they reach your team. When you change a prompt, swap Claude models, or update your ICP taxonomy, this suite replays a set of known-correct inputs through Claude and scores the outputs against four criteria. If anything regresses, it fails loudly - in your terminal and optionally in Slack - before the bad output ever appears in a weekly report.

This directive assumes `directives/langfuse_observability.md` has already been implemented. The eval layer builds on top of the Langfuse client and trace infrastructure created there.

---

## When This Runs
- **Manually** - before any significant prompt change, model upgrade, or ICP taxonomy update
- **Automatically in CI** - via GitHub Actions on every commit that touches `execution/analyze.py`, any file in `directives/`, or any file in `config/`
- **Not** part of the weekly scheduled run

---

## Prerequisites

- `directives/langfuse_observability.md` fully implemented (Langfuse client, `call_claude` wrapper, `.env` keys in place)
- At least one completed weekly production run so you have real outputs to promote to datasets

---

## Step 1: Create Datasets in Langfuse

Build `scripts/setup_langfuse_datasets.py`. This is a one-time setup script - run it once after your first successful weekly run.

**What it must do:**
Call `langfuse.create_dataset(name=..., description=...)` for each of the following datasets:

| Dataset Name | Purpose |
|---|---|
| `doe-keyword-intelligence` | Golden examples for Call 1 |
| `doe-lead-scoring` | Golden examples for Call 2 |
| `doe-competitor-audit` | Golden examples for Call 3 |
| `doe-cross-vertical-summary` | Golden examples for Call 11 |

If a dataset already exists, skip it without error.

Print the name of each dataset created or skipped. Call `langfuse.flush()` before the script exits.

**How to run:**
```bash
python scripts/setup_langfuse_datasets.py
```

---

## Step 2: Seed Datasets with Golden Examples

Add a `promote_to_dataset` function to `execution/langfuse_client.py`.

**Function signature:**
```python
def promote_to_dataset(
    *,
    dataset_name: str,
    system: str,
    messages: list[dict],
    model: str,
    expected_output: str,
    metadata: dict = None,
) -> str:
    """
    Saves a production input+output pair as a golden dataset item.
    Returns the created item ID.
    """
```

**What it must do:**
1. Call `langfuse.get_dataset(dataset_name).create_item(input={...}, expected_output=expected_output, metadata=metadata or {})`
2. The `input` dict must contain: `{"system": system, "messages": messages, "model": model}`
3. Return the item ID

**How to use it in `analyze.py`:**

After any Claude call produces an output you have reviewed and confirmed is correct, call `promote_to_dataset` once (manually, then comment it out):

```python
# Seed once - comment out after running
from langfuse_client import promote_to_dataset

promote_to_dataset(
    dataset_name="doe-keyword-intelligence",
    system=SYSTEM_PROMPT,
    messages=keyword_messages,
    model=MODEL_DEFAULT,
    expected_output=keyword_text,
    metadata={"week": run_date, "vertical_count": 2},
)
```

**Target:** 3-5 dataset items per call, covering meaningfully different scenarios. For example:
- A week with at least one zero-conversion keyword alert
- A week with a CRITICAL urgency flag in the output
- A week where GBP data was missing and the output handled it gracefully
- A normal week with no alerts (the baseline)

Diversity matters more than volume. Four varied items catch more regressions than ten similar ones.

---

## Step 3: Build the Scorers

Build `execution/evals/scorers.py`. This file contains four scorer functions. Each returns a `tuple[float, str]` - a score between 0.0 and 1.0, and a human-readable reason string.

**Scorer 1 - Valid urgency labels**
- Regex-scan the output text for all `"urgency": "..."` patterns
- If any value is not one of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, return `(0.0, "Invalid urgency labels found: {...}")`
- If no urgency labels are found at all, return `(0.5, "No urgency labels found")`
- If all labels are valid, return `(1.0, "All urgency labels valid")`

**Scorer 2 - Required fields present**
- Attempt to parse the output as JSON
- If parsing fails, return `(0.0, "Output is not valid JSON")`
- Check that `section_title`, `insights`, and `raw_text` all exist as top-level keys
- If any are missing, return `(0.0, f"Missing required fields: {missing}")`
- If all present, return `(1.0, "All required fields present")`

**Scorer 3 - No hallucinated dollar figures**
- Extract all `$[digits]` patterns from the output
- Extract all digit sequences from the input (the raw CSV data sent to Claude)
- For each dollar figure in the output, check whether its numeric value appears in the input
- If any output dollar figure has no match in the input, return `(0.0, f"Dollar figures not found in input: {list[:5]}")`
- If all match, return `(1.0, "All cited figures traceable to input")`

**Scorer 4 - Output length stability**
- Compare `len(output_text)` to `len(expected_output)` (the golden example from the dataset)
- If the ratio differs by more than 40%, return `(0.0, f"Output length changed by {pct}% vs baseline")`
- Otherwise return `(1.0, "Output length within 40% of baseline")`

---

## Step 4: Build the Regression Test Runner

Build `execution/evals/run_evals.py`. This is the main eval script - it replays every dataset item through Claude and scores the result.

**What it must do:**

1. Initialize the Anthropic client and import `langfuse` from `langfuse_client`
2. Set `RUN_NAME = f"regression-{os.environ.get('GIT_SHA', 'local')[:8]}"` - this identifies the run in Langfuse
3. For each dataset in `DATASET_NAMES`:
   - Call `langfuse.get_dataset(dataset_name)` to get all items
   - For each item, run `run_single_item(dataset_name, item)`
4. At the end, print a summary: total passed, total failed
5. Call `langfuse.flush()` before exiting
6. Exit with code `1` if any item failed, `0` if all passed - this makes CI fail on regressions

**`run_single_item` must:**
1. Extract `model`, `system`, `messages` from `item.input`
2. Call `client.messages.create(...)` directly (not via `call_claude` - this avoids creating production traces during eval runs)
3. Run all four scorers on the output
4. Create a Langfuse trace observation named `f"{dataset_name}/{RUN_NAME}"` with `session_id=RUN_NAME` using `with langfuse.start_as_current_observation(as_type="trace", name=..., session_id=...) as trace_obs:`
5. Log a generation inside the trace context (input, output, token usage)
6. Attach all four scores using `langfuse.score(trace_id=trace_obs.trace_id, name=..., value=..., comment=...)`
7. Return a dict: `{"item_id": item.id, "scores": {...}, "passed": bool}`

**Score names to use** (must match exactly - Langfuse uses these as metric keys in the dashboard):
- `urgency_labels_valid`
- `required_fields`
- `no_hallucinated_numbers`
- `output_length_stable`

**How to run:**
```bash
python execution/evals/run_evals.py
```

---

## Step 5: Add GitHub Actions CI

Build `.github/workflows/doe_evals.yml`.

**Trigger conditions:**
- On push to any branch when the following paths change:
  - `execution/analyze.py`
  - `directives/**`
  - `config/**`
- On `workflow_dispatch` (manual trigger from GitHub UI)

**Job steps:**
1. Checkout the repo
2. Set up Python 3.12
3. `pip install anthropic langfuse`
4. Run `python execution/evals/run_evals.py`

**Environment variables to inject from GitHub Secrets:**
- `ANTHROPIC_API_KEY`
- `LANGFUSE_SECRET_KEY`
- `LANGFUSE_PUBLIC_KEY`
- `LANGFUSE_HOST`
- `GIT_SHA` - set this to `${{ github.sha }}` so each CI run is uniquely identified in Langfuse

**GitHub Secrets to add** (go to repo → Settings → Secrets and variables → Actions):
- `ANTHROPIC_API_KEY` - your Anthropic key
- `LANGFUSE_SECRET_KEY` - from Langfuse project settings
- `LANGFUSE_PUBLIC_KEY` - from Langfuse project settings
- `LANGFUSE_HOST` - `https://us.cloud.langfuse.com` or `https://cloud.langfuse.com`

> Note: These eval runs make real Claude API calls. Budget approximately $0.10-$0.30 per CI run depending on how many dataset items you have seeded. Keep your dataset small (3-5 items per call) until you understand your CI cost.

---

## What Each Scorer Catches

| Scorer | What regression it catches |
|---|---|
| `urgency_labels_valid` | Prompt change causes Claude to invent urgency values not defined in the system prompt (e.g. `"URGENT"` instead of `"CRITICAL"`) |
| `required_fields` | Prompt change silently drops a required JSON key or breaks JSON formatting entirely |
| `no_hallucinated_numbers` | Model change causes Claude to cite dollar figures that do not exist in the input CSV data |
| `output_length_stable` | Silent truncation (max_tokens set too low) or verbosity explosion from a new model version |

---

## Where to See Results

**In your terminal:**
```
--- Running evals: doe-keyword-intelligence ---
  item a3f1bc2d: ✅ PASS - {'urgency_labels_valid': 1.0, 'required_fields': 1.0, ...}
  item 9e22a1f0: ❌ FAIL - {'required_fields': 0.0, ...}

=== Eval Summary ===
Passed: 7  |  Failed: 1

❌ Regression detected - review failures in Langfuse before merging.
```

**In Langfuse (`/datasets`):**
- Click any dataset → Items tab → see every eval run side by side
- Scores are displayed per item per run - immediately shows which run introduced the regression

**In Langfuse (`/dashboard → Scores`):**
- Charts of score pass rates across all CI runs over time
- Filter by score name to isolate which scorer is failing

---

## Recommended Eval Cadence

| Trigger | Action |
|---|---|
| Any edit to `analyze.py` or a directive | CI runs automatically |
| Anthropic releases a new Claude version | Run manually + review 2-3 outputs in Langfuse before switching models in `.env` |
| ICP taxonomy changes (`config/icp_taxonomy.py`) | Re-seed `doe-lead-scoring` with a fresh golden example, then run the suite |
| Weekly production run | No eval run needed - observability traces are captured automatically via `langfuse_observability.md` |

---

## Edge Cases and Notes

- **Empty datasets:** If a dataset has no items yet (not yet seeded), the runner should skip it with a warning rather than failing. Datasets are only useful once you have real golden examples.
- **Dataset item format:** The `item.input` dict must always contain `system`, `messages`, and `model`. If an item is missing any of these keys, skip it and log a warning.
- **CI cost control:** If CI costs are a concern, add a `--dry-run` flag that skips the Claude API call and only validates that datasets are populated. This lets you verify CI configuration without incurring API costs.
- **eval/ as a package:** Add an empty `execution/evals/__init__.py` so Python treats it as a package and imports work correctly.

---

## Output Files

No files are written to `.tmp/`. All output goes to Langfuse.

---

## Scripts This Directive Feeds

- `execution/langfuse_client.py` - **[MODIFY]** Add `promote_to_dataset` function
- `execution/evals/__init__.py` - **[NEW]** Empty package init
- `execution/evals/scorers.py` - **[NEW]** Four scorer functions
- `execution/evals/run_evals.py` - **[NEW]** Regression test runner
- `scripts/setup_langfuse_datasets.py` - **[NEW]** One-time dataset creation
- `.github/workflows/doe_evals.yml` - **[NEW]** CI workflow
