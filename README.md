# PagerDuty for GitHub Copilot

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's operational intelligence plugin for the [GitHub Copilot](https://github.com/features/copilot) AgentHQ plugin system. Brings PagerDuty incident data and Skills management into your GitHub workflow.

## Capabilities

| Skill | What it does |
| --- | --- |
| `create-pagerduty-skill` | Create or update PagerDuty skills for AI agents through guided interview |
| `investigate` | Multi-turn SRE Agent investigation for a PagerDuty incident |
| `on-call-handoff` | Generate an on-call incident summary or shift handoff using PagerDuty data |
| `pre-commit-risk-scoring` | Assess pre-commit risk by correlating PagerDuty incidents with current code changes |

## Prerequisites

- A working [GitHub Copilot CLI](https://docs.github.com/copilot) installation
- A PagerDuty API token, exposed to the harness as `PAGERDUTY_API_KEY` (see [Where to get a token](#where-to-get-a-pagerduty-api-token))
- For `create-pagerduty-skill`: enrollment in both [PagerDuty Skills EA](https://www.pagerduty.com/early-access/) and PagerDuty Advance MCP/API EA

## Install

```bash
copilot plugin install PagerDuty/github-agenthq-plugins
```

Verify:

```bash
copilot plugin list                 # should list PagerDuty/github-agenthq-plugins
# In an interactive Copilot session:
/agent                              # confirms the PagerDuty main agent loaded
/skills list                        # confirms both skills loaded
```

## Usage

Once installed and wrapped in an Agentic App, the plugin is invokable via the App's GitHub identity:

- **@-mention** the App in an issue, PR, or discussion to start a session
- **Assign** an issue to the App for autonomous triage
- **Trigger** it from Mission Control

### Triggering pre-commit-risk-scoring

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

### Triggering create-pagerduty-skill

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

## Authentication

Both skills authenticate to PagerDuty via `PAGERDUTY_API_KEY`. The token is passed as `Authorization: Token token=${PAGERDUTY_API_KEY}` on every MCP request.

### Where to get a PagerDuty API token

PagerDuty web UI → User Profile → User Settings → API Access → **Create API User Token**.

## Reporting issues

File bugs and feature requests at https://github.com/PagerDuty/github-agenthq-plugins/issues. Security concerns go to **security@pagerduty.com** — see [SECURITY.md](SECURITY.md).

## License

Apache 2.0 — see [LICENSE](LICENSE).
