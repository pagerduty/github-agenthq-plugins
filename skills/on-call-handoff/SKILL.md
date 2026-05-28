---
name: on-call-handoff
description: Generate an on-call incident summary or shift handoff using PagerDuty data
---

Generate an on-call incident summary or shift handoff from PagerDuty data. Follow these steps in order. Do not skip steps. Do not silently degrade — if a required tool is unavailable, stop and tell the user.

Arguments from the user: `[team_name] [timeframe]`. Both are optional.
- **team_name** — a non-numeric string with no time unit (e.g. `platform`, `payments`). Doubles as the config cache key.
- **timeframe** — a duration expression containing a number and unit (e.g. `7 days`, `2 weeks`, `30d`). Defaults to the escalation policy's `rotation_duration`, falling back to 7 days.

Parse by scanning for a time-unit pattern first; treat any remaining tokens as `team_name`.

## Pre-flight

Before doing anything else, verify PagerDuty connectivity.

Call `pagerduty-list_services` with `query: "test"`. This is a connectivity check.

If the call returns a transport, auth, or "tool not found" error, STOP and tell the user:

```
PagerDuty MCP server is not available. This skill requires it.

The Agentic App configuration needs COPILOT_MCP_PAGERDUTY_API_KEY in its environment.
Check the app's settings, then retry once the variable is set.
```

Do NOT proceed without PagerDuty MCP.

## Step 1 — Scope resolution

**Check cache:** read `~/on-call-handoff/config.json` via `bash`. Look for a profile key matching `team_name` (if provided).

- If a matching profile is found and contains non-empty `escalation_policies` and `services`, use it. Skip to "Confirm scope" below.
- Otherwise: resolve fresh from PagerDuty.

**Resolve from PagerDuty (run in parallel where possible):**
1. Call `pagerduty-get_user_data` to get the current user's email.
2. Call `pagerduty-list_escalation_policies` to list policies. If `team_name` was provided, filter by name match. For each matching policy, call `pagerduty-get_escalation_policy` to get its schedules; for each schedule call `pagerduty-get_schedule` to retrieve `rotation_duration`.
3. Call `pagerduty-list_services` filtered to current user.

**Profile key:** use `team_name` if provided, otherwise the user's email.

**Timeframe:** use `timeframe` from the user's message if provided. Otherwise use the first escalation policy's `rotation_duration`; fall back to 7 days if unavailable.

**Save cache:** after resolving, write the profile to `~/on-call-handoff/config.json` via `bash`, merging with any existing profiles:
```json
{
  "[profile-key]": {
    "escalation_policies": [{"id": "...", "name": "...", "rotation_duration": "..."}],
    "services": [{"id": "...", "name": "..."}]
  }
}
```
If the write fails, note the error and continue — do not block the report.

**Confirm scope:** tell the user which escalation policy and timeframe you resolved. Ask them to confirm or correct before proceeding.

Do not proceed to Step 2 until the user has confirmed scope and timeframe.

## Step 2 — Data gathering

Run all calls in parallel:

1. **Active incidents** — call `pagerduty-list_incidents` with `statuses[]=triggered,acknowledged` scoped to the services in the resolved profile.

2. **All incidents in window** — call `pagerduty-list_incidents` with `since`/`until` filters covering the timeframe, for the same service scope. Include all statuses.

3. **Stale check** — from call 1, flag any triggered incident open >7 days.

From the collected data derive:
- **Top services by incident volume** (from call 2)
- **Recurring alert patterns** — deduplicate incidents by title; a title appearing ≥3× in the window is a recurring pattern
- **Volume signal** — flag if total incident count is >50 for a 7-day timeframe (or proportionally adjusted)
- **API cap note** — if call 2 returns exactly 1000 incidents, the history is partial; note the oldest incident's timestamp

## Step 3 — Output

Adapt output style to timeframe: ≤14 days → handoff-focused (Active first); >14 days → trend-focused (Incident Trends first).

```
## On-Call Summary — [Escalation Policy Name]
Period: [start] → [end] | Generated: [date]

### Active  [omit if no active incidents]
[triggered/acknowledged incidents — one line each: ID, service, age, assignee]

### Incident Trends
Top services: [service — N incidents] (one line each, top 5 max)
Recurring alerts: [title — N occurrences] (or "None")
Volume: [N incidents in period; note if unusually high]

### Watch List
[stale incidents (open >7d), services with sharp volume spikes, recurring alert patterns needing attention]

### Recommended Actions
[3–5 concrete items, most urgent first]
```

Keep it scannable — one line per item. Omit any section that has nothing to report.

## Step 4 — Follow-up

Call `ask_user` with:
- `question`: "Found [N] incidents in [timeframe]. What would you like to do next?"
- `choices`: ["Deep analysis with SRE Agent", "Save to file", "Done"]

Handle each response:

- **Deep analysis with SRE Agent** — present the incidents from the report as a numbered list using `ask_user` with choices: show triggered/acknowledged incidents first, then notable watch list items, up to 5 total (include incident number, truncated title, service, and age in each choice label). Include a final choice: **"List all incidents"** — if selected, output every incident from the handoff window as a numbered list (`#[number] [title] — [service] — [age]`) and call `ask_user` again asking which one they want. Once an incident is selected by any path, invoke the `sre-investigate` skill immediately by calling the `skill` tool with `skill: "sre-investigate"`, then follow that skill's instructions using the selected incident ID.
- **Save to file** — write the summary to `./on-call-reports/[escalation-policy-slug]_[YYYY-MM-DD_HH-MM].md` via `bash`. Create `./on-call-reports/` if it does not exist. Confirm the path to the user.
- **Done** — finish without saving.
