---
name: investigate
description: Multi-turn SRE Agent investigation for a PagerDuty incident
---

Investigate a PagerDuty incident with the SRE Agent in a multi-turn conversation. Follow these steps in order. Do not silently degrade — if a required tool is unavailable, stop and explain.

The user will provide an incident ID (e.g. `Q3AHWAR33KUK3G` or `#971097`), or ask you to list active incidents first.

## Pre-flight

Call `pagerduty-list_incidents` with `limit=1` to confirm the PagerDuty MCP server is connected.

If the call fails with a transport, auth, or "tool not found" error, STOP and tell the user:

```
PagerDuty MCP server is not available. This skill requires it.

The Agentic App configuration needs COPILOT_MCP_PAGERDUTY_API_KEY in its environment.
Check the app's settings, then retry once the variable is set.
```

Do NOT proceed without PagerDuty MCP.

## Step 1 — Incident resolution

Parse the user's message:

**Alphanumeric ID** (e.g. `Q3AHWAR33KUK3G`): use directly as `incident_id`. Skip to Step 2.

**Numeric or `#`-prefixed** (e.g. `#971097`): strip `#`, call `pagerduty-list_incidents` and find the incident whose `incident_number` matches. Use its `id` field. If no match, ask the user to provide the alphanumeric ID directly.

**No ID given**: call `pagerduty-list_incidents` with `status=[triggered,acknowledged]`. Show the first 10 results (ID, title, service, age) and ask which one to investigate. If empty, tell the user no active incidents were found and ask them to provide an incident ID or describe what they want to investigate.

Do not call the SRE Agent until an incident ID is confirmed.

## Step 2 — Initial analysis

Call `pagerduty-advance-sre_agent_tool` with:
- `incident_id`: the resolved alphanumeric ID
- `message`: `"Summarize incident [id]: what happened, root cause, and top recommended actions."`

Do **not** pass `session_id` on this first call.

Store the `session_id` from the response — reuse it in every follow-up call. Never start a new session mid-conversation.

Display the SRE Agent's full response.

If the call times out, retry once with `"Summarize incident [id]."`. If it times out again, tell the user the SRE Agent is unavailable and stop.

## Step 3 — Conversation loop

After displaying each SRE Agent response, call `ask_user` with:
- `question`: "What would you like to explore next?"
- `choices`: ["Save to file", "Done"]
- `allow_freeform`: true (the user can type a follow-up question directly)

Handle each response:

**Follow-up question** (freeform input): call `pagerduty-advance-sre_agent_tool` with `incident_id`, `message=<user question>`, and `session_id=<stored>`. Display the result. Ask the same follow-up prompt again.

**Save to file**: write the full conversation transcript to `./on-call-reports/sre-[incident-id]_[YYYY-MM-DD_HH-MM].md` via `bash`. Create `./on-call-reports/` if needed. Confirm the path. Continue the loop — the user may ask more questions after saving.

**Done**: exit.
