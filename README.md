# Demand Stack Automation

Two open-source tools for teams using [Claude Code](https://claude.ai/code) to run serious, multi-session projects.

---

## What's in here

### 1. Global Skill (`GLOBAL_SKILL.md`)
A drop-in behavioral layer for Claude Code that makes it better at long projects and cleaner at writing code. Install it once globally and it applies to every project.

**Skill 1: Session Relay** *(Credit: [Andy Tozier](https://github.com/Andytoizer/session-relay), Head of Growth @ [Freckle.io](https://freckle.io/) | Writing [AgentOperator](https://agentoperator.substack.com/) (building GTM with agentic tools))*
Manages context across long projects so nothing is lost between sessions. Teaches Claude Code when to compact vs. start a new session, how to maintain a README as the project source of truth, and how to generate clean handoff prompts between sessions.

**Skill 2: Karpathy Coding Guidelines**
Reduces common LLM coding mistakes. Enforces minimal, surgical, assumption-free code — no speculative features, no unnecessary abstractions, no silent assumptions.

**Install:**
```bash
cp GLOBAL_SKILL.md ~/.claude/CLAUDE.md
```
If you already have a `~/.claude/CLAUDE.md`, append the contents rather than overwriting.

---

### 2. DOE Framework (`doe-framework/`)
A fully automated weekly marketing intelligence system for B2B SaaS companies. It pulls data from your ad platforms, CRM, search tools, Reddit, and LinkedIn, runs it through Claude for analysis, and delivers a structured PDF report to Slack every week.

The framework uses a 3-layer architecture:
- **Directives** — SOPs in `doe-framework/directives/` that define exactly what to pull, from where, and how to handle edge cases
- **Orchestration** — Claude Code acts as the decision-maker, reading directives and calling execution scripts
- **Execution** — Deterministic Python scripts that do the actual data pulling and processing. Cheaper to run during production; python scripts all run for free. The only Claude API usage is when analyzing and generating the weekly report.

Includes 12 directive templates covering keyword intelligence, CRM attribution, paid lead scoring, competitor auditing, Reddit mining, LinkedIn organic, conversion funnel analysis, meeting booking, email sequences, Google Business Profile, entity optimization, and weekly report generation.

**[See `doe-framework/README.md` for full setup instructions →](doe-framework/README.md)**

---

## License

MIT
