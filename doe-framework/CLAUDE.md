# [YOUR_COMPANY] Marketing Intelligence Automation

## Project Overview
This automation runs on a weekly schedule and generates a marketing intelligence report delivered to [YOUR_SLACK_CHANNEL] tagging [TEAM_MEMBER_1], [TEAM_MEMBER_2], and [TEAM_MEMBER_3].

## The DOE Framework
DOE stands for Directives → Orchestration → Execution. Directives are SOPs that define *what* data to pull and *why*. The Orchestration layer (Claude Code) makes routing decisions, handles errors, and calls execution scripts in order. Execution scripts are deterministic Python files that pull data from APIs and save outputs to `.tmp/`. The orchestrator never touches raw data directly — it delegates to execution scripts and reads their outputs.

## Layers
- **Layer 1 (Directives):** What to do — SOPs in `directives/`
- **Layer 2 (Orchestration):** Decision-making — Claude Code routes and handles errors
- **Layer 3 (Execution):** Deterministic Python scripts in `execution/`

## Rules
1. Always read the relevant directive BEFORE writing or running any script
2. When something breaks: fix it, test it, update the directive
3. Never delete files without asking
4. Never commit `.env` to GitHub
5. When ambiguous, ask — don't assume
6. Update `README.md` at phase boundaries before compacting or ending a session

## Verticals
<!-- Replace with your own ICP verticals. -->
<!-- Example format: -->
<!-- Primary: [Vertical 1], [Vertical 2] -->
<!-- Secondary: [Sub-vertical A], [Sub-vertical B], [Sub-vertical C] -->

Vertical 1: [VERTICAL_1_NAME] — [brief description]
Vertical 2: [VERTICAL_2_NAME] — [brief description]

## Delivery
Slack channel: [YOUR_SLACK_CHANNEL]
Tag: [TEAM_MEMBER_1] <@[SLACK_USER_ID_1]>, [TEAM_MEMBER_2] <@[SLACK_USER_ID_2]>, [TEAM_MEMBER_3] <@[SLACK_USER_ID_3]>
Schedule: Every Sunday [TIME] via [SCHEDULER] (Modal, cron, etc.)

## Self-Annealing Loop
When errors occur:
1. Fix the script
2. Test it
3. Update the directive
4. Move on

## File Organization
- `.tmp/` — temporary files during processing, never commit
- `execution/` — all Python scripts
- `directives/` — all SOP markdown files
- `config/` — ICP taxonomy and competitor lists
- `.env` — all API keys, never share or commit
