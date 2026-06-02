# PagerDuty for GitHub Copilot

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's operational intelligence plugin for the [GitHub Copilot](https://github.com/features/copilot) AgentHQ plugin system. Brings PagerDuty incident data and Skills management into your GitHub workflow.

[Install PagerDuty Github App](https://github.com/apps/pagerduty-agent-app)

## Capabilities

| Skill | What it does |
| --- | --- |
| [`create-pagerduty-skill`](#create-pagerduty-skill) | Create or update PagerDuty skills for AI agents through guided interview |
| [`investigate`](#investigate) | Multi-turn SRE Agent investigation for a PagerDuty incident |
| [`on-call-handoff`](#on-call-handoff) | Generate an on-call incident summary or shift handoff using PagerDuty data |
| [`pre-commit-risk-scoring`](#pre-commit-risk-scoring) | Assess pre-commit risk by correlating PagerDuty incidents with current code changes |

## Prerequisites

- A working [GitHub Copilot CLI](https://docs.github.com/copilot) installation
- A PagerDuty API token, exposed to the harness as `COPILOT_MCP_PAGERDUTY_API_KEY` (see [Where to get a token](#where-to-get-a-pagerduty-api-token))
- For `create-pagerduty-skill`: enrollment in both [PagerDuty Skills EA](https://www.pagerduty.com/early-access/) and PagerDuty Advance MCP/API EA
- For `investigate`: access to PagerDuty Advance MCP (`PAGERDUTY_API_KEY` must grant Advance access)

## Install

```bash
copilot plugin install PagerDuty/github-agenthq-plugins
```

Verify:

```bash
copilot plugin list                 # should list PagerDuty/github-agenthq-plugins
# In an interactive Copilot session:
/agent                              # confirms the PagerDuty main agent loaded
/skills list                        # confirms all skills loaded
```

## create-pagerduty-skill

*Create or update PagerDuty Skills for the SRE Agent.*

Ask the agent to create or update a PagerDuty Skill:

> "Create a new PagerDuty skill"
> "Update my `<skill-name>` skill"

The skill walks you through:

1. Create vs. update
2. Scope selection (personal user vs. shared account)
3. Workflow description in your own words
4. Auto-drafted, structured instructions for the SRE Agent
5. Name + description suggestions, validated against API constraints
6. Immediate deployment to the PagerDuty platform

## investigate

*Multi-turn SRE Agent investigation with session continuity.*

Ask the agent to investigate a PagerDuty incident:

> "Investigate incident Q3AHWAR33KUK3G"
> "What caused incident #971097?"
> "Root cause analysis for this incident"

The skill opens a multi-turn SRE Agent session:

- Resolves the incident ID (alphanumeric or numeric; lists active incidents if none provided)
- Returns a structured summary: what happened, root cause, and recommended actions
- Maintains session context across follow-up questions — deployment correlation, similar past incidents, log links — without reloading context each turn
- Optionally saves the full conversation transcript to `./on-call-reports/sre-<id>_<date>.md`

Requires PagerDuty Advance MCP access (see [Prerequisites](#prerequisites)).

## on-call-handoff

*On-call incident summary and shift handoff.*

Ask the agent for an on-call summary or shift handoff:

> "Give me an on-call handoff for the payments team"
> "Summarize the last 7 days of incidents"
> "What happened on-call this week?"

Optional arguments: `[team_name] [timeframe]` — e.g. `payments 7 days`, `platform 2 weeks`.

The skill:

- Resolves your team's escalation policies and services from PagerDuty (cached after first run in `~/on-call-handoff/config.json`)
- Fetches active incidents and all incidents in the window in parallel
- Returns a structured summary: Active, Incident Trends, Watch List, Recommended Actions
- Adapts format to timeframe: ≤14 days → handoff-focused (Active first), >14 days → trend-focused (Incident Trends first)
- Offers to save to `./on-call-reports/<team>_<date>.md` or bridge to `investigate` for deep analysis on a specific incident

## pre-commit-risk-scoring

*Pre-commit risk assessment correlated with PagerDuty incident history.*

Ask the agent about risk before committing or pushing:

> "Run a pre-commit risk check on this branch"
> "Is this change risky?"
> "Compare these changes to recent PagerDuty incidents on `<service-name>`"

The skill resolves your repo to a PagerDuty service through a fallback chain (explicit argument → cached config → Backstage `catalog-info.yaml` annotation → repo-name auto-detection → user prompt) and returns:

- Active incidents on the mapped service
- 90-day incident history summary
- File-level and structural risk signals in the diff
- Correlation findings between current changes and past incidents
- A risk score (0 = no risk, 5 = critical) with recommendations

## Authentication

All skills authenticate to PagerDuty via `PAGERDUTY_API_KEY`. The token is passed as `Authorization: Token token=${PAGERDUTY_API_KEY}` on every MCP request.

### Where to get a PagerDuty API token

PagerDuty web UI → User Profile → User Settings → API Access → **Create API User Token**.

## Reporting issues

File bugs and feature requests at https://github.com/PagerDuty/github-agenthq-plugins/issues. Security concerns go to **security@pagerduty.com** — see [SECURITY.md](SECURITY.md).

## License

Apache 2.0 — see [LICENSE](LICENSE).
