---
name: pre-commit-risk-scoring
description: Assess pre-commit risk by correlating PagerDuty incidents with current code changes
---

You are performing a pre-commit risk assessment. Follow these steps precisely and in order. Do not skip steps. Do not silently degrade — if a required tool is unavailable, stop and tell the user.

## Step 0: Pre-flight checks

Before doing anything else, verify that the required MCP tools are available.

### 0a-pre: Determine mode

Before anything else, check how this skill was invoked:

1. If `GITHUB_ACTIONS=true` is set: set `BRANCH_MODE=true`. If `GITHUB_BASE_REF` is non-empty, set `BASE_BRANCH=$GITHUB_BASE_REF` and skip Steps 0b and 0c entirely. Otherwise proceed to Step 0c for base branch detection. Treat any user text as the PagerDuty service name (may be empty).
2. If the user explicitly passed `--branch`, mentioned "assess the branch", "check the branch", or otherwise asked for a branch-level assessment: set `BRANCH_MODE=true`. Treat any remaining text as the PagerDuty service name (may be empty — that is fine).
3. Otherwise: set `BRANCH_MODE=false`.

Carry `BRANCH_MODE` and `BASE_BRANCH` forward through all subsequent steps.

### 0a: Verify PagerDuty MCP connectivity

Call `pagerduty-list_services` with `query: "test"`. This is a connectivity check.

If the call returns a transport, auth, or "tool not found" error, STOP and tell the user:

```
PagerDuty MCP server is not available. This skill requires it.

The Agentic App configuration needs COPILOT_MCP_PAGERDUTY_API_KEY in its environment.
Check the app's settings, then retry once the variable is set.
```

Do NOT proceed without PagerDuty MCP. Do NOT fall back to a degraded assessment — the entire point of this skill is PagerDuty incident correlation.

### 0b: Determine assessment mode

If `BRANCH_MODE=true` (set in Step 0a-pre):
- Skip directly to **Step 0c: Base branch detection**. Do not check for uncommitted changes.

If `BRANCH_MODE=false`:
- Run `git diff --stat` and `git diff --cached --stat` via `bash`.
- **If either has output** (uncommitted changes exist): proceed with the existing uncommitted-changes assessment (Steps 1–6). After presenting the final assessment in Step 6, offer the branch follow-up as described in **Step 6b** below.
- **If both are empty**: auto-switch to branch mode.
  - Run `git rev-parse --abbrev-ref HEAD` to get the current branch name.
  - If the current branch is `main` or `master`, tell the user:

    ```
    No uncommitted changes detected, and you're already on the base branch (main/master).
    There is nothing to assess.
    ```

    Then STOP.
  - Otherwise: set `BRANCH_MODE=true` and continue to **Step 0c: Base branch detection**. Tell the user briefly: "No uncommitted changes found — switching to branch mode."

### 0c: Base branch detection (branch mode only)

This step runs only when `BRANCH_MODE=true`.

Detect the base branch by trying the following in order — stop at the first that resolves cleanly:

1. **GitHub MCP (agent context):** If `GITHUB_ACTIONS=true`, get the repo owner and name from `git remote get-url origin` via `bash` (parse from SSH or HTTPS URL). Call `github-mcp-server-list_pull_requests` with `owner`, `repo`, `head` set to `<owner>:<current-branch>`, and `state: "open"`. If one or more PRs are returned, use the `base.ref` field as the base branch. Do NOT use `gh` CLI in this context — `api.github.com` is blocked by the agent firewall.
2. **GitHub CLI (local context):** If `GITHUB_ACTIONS` is not set, run `gh pr view --json baseRefName --jq '.baseRefName' 2>/dev/null` via `bash`. If it returns a non-empty string, that is the base branch.
3. **Upstream tracking branch:** Run `git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null` via `bash`. If it returns something like `origin/main`, strip the remote prefix (`origin/`) to get the base branch name.
4. **Remote branch scan:** Check which of `main`, `master`, `develop` exist as remote branches: `git branch -r | grep -E 'origin/(main|master|develop)$'`. Use the first match found.
5. **Ask the user:** If none of the above resolve, ask the user conversationally: "Which branch should I compare against? (e.g. main, master, develop)"

**Guard — current branch IS the base:** After resolving the base, run `git rev-parse --abbrev-ref HEAD` to get the current branch name. If it matches the base branch name, tell the user:

```
You're already on the base branch (<base>) — there's no branch diff to assess.
```

Then STOP.

**Guard — no commits ahead of base:** Run `git log <base>..HEAD --oneline` via `bash`. If the output is empty, tell the user:

```
No commits found ahead of <base> — nothing to assess in branch mode.
```

Then STOP.

Store the resolved base branch name as `BASE_BRANCH` for use in Steps 4a and 6.

## Step 1: Resolve service mapping

Determine the PagerDuty service ID for this repository. Follow the steps **in order** and stop at the first one that resolves a service.

### 1a: Use explicit argument (highest priority)

If the user passed a service name or ID with the invocation (the cleaned value from Step 0a-pre, after stripping `--branch` if present), use it immediately — **do not check the cache or catalog first**.

Call `pagerduty-list_services` with the user-supplied value as the query.

- Exactly one match → use it and skip to Step 1d.
- Multiple matches → list them numbered to the user, ask which one, then proceed.
- No match → tell the user no service was found for that name, ask them to verify, and STOP. Do NOT fall through to cache or repo-name detection.

### 1b: Check cached configuration

If no explicit argument was provided, read `.pagerduty/pagerduty-risk-config.json` via `view`. If it exists and contains `pagerduty.serviceId`, validate it by calling `pagerduty-get_service` with that ID.

- Call succeeds → use the service, display its name to the user, and skip to Step 2.
- Call fails (service not found or API error) → discard the cached config, warn the user, and continue to Step 1c.

### 1c: Check Backstage catalog

Check for `catalog-info.yaml` in the repository root using `view`. Look for the `pagerduty.com/service-id` annotation under `metadata.annotations`. If found, use that service ID.

Validate by calling `pagerduty-get_service` with the literal service ID (e.g. `PAWX771`). **Do NOT pass the ID to `pagerduty-list_services`** — querying by raw UUID returns a 502. Extract the service name from the response. If found, skip to Step 1d.

### 1d: Auto-detect from repository name

If none of the above resolved a service:

- Get the repository name from the current directory basename, or parse it from `git remote -v` output.
- Call `pagerduty-list_services` with a query matching the repository name.
- Exactly one match → use it. Tell the user which service was detected and ask them to confirm before proceeding.
- Multiple matches → list them numbered, ask the user to pick.
- No match → ask the user for the PagerDuty service name or ID directly. Validate the response with `pagerduty-list_services`.

### 1e: Persist configuration

If the service was resolved via the explicit argument (Step 1a), **do not write or update `.pagerduty/pagerduty-risk-config.json`** — the argument is a one-time override, not a permanent mapping.

Otherwise, once a service ID is resolved, write `.pagerduty/pagerduty-risk-config.json` (create the `.pagerduty/` directory first if it does not exist):

```json
{
  "version": "1.0",
  "pagerduty": {
    "serviceId": "<resolved-service-id>",
    "serviceName": "<resolved-service-name>"
  }
}
```

## Step 2: Check ongoing incidents

Call `pagerduty-list_incidents` with:

- `statuses`: `["triggered", "acknowledged"]`
- `service_ids`: `["<serviceId>"]`

For each active incident returned, call `pagerduty-list_incident_notes` for responder context. **Cap at 5 active incidents** — if more are returned, fetch notes only for the 5 most recently triggered and note the total count.

If there are active incidents, report them prominently — they are the highest-priority risk signal:

```
ACTIVE INCIDENTS

[TRIGGERED] INC-12345: Database connection pool exhaustion (P1)
  Triggered 2h ago. Notes: "Scaling up read replicas, ETA 30min"

[ACKNOWLEDGED] INC-12346: Elevated error rate on /api/checkout (P2)
  Acknowledged 45min ago. Notes: "Investigating correlation with deploy at 14:30"
```

If there are no active incidents, note that briefly and continue.

## Step 3: Fetch recent incident history

**Run these calls in parallel with Step 2** — they are fully independent. Three calls in parallel:

1. `pagerduty-list_incidents` for the service over the last 90 days, filtered to **high urgency** (`since` 90 days ago, `until` today, `urgencies: ["high"]`). Include all statuses.

   **Limit handling:** the API caps results at 1000 incidents. If exactly 1000 are returned, the history is partial — note this in the assessment and state the actual date range covered (timestamp of the oldest incident returned).

2. `pagerduty-list_incidents` for the service over the last 90 days, filtered to **low urgency** (same date range, `urgencies: ["low"]`). Include all statuses. 

    Apply the same limit handling as above independently for this call.

3. `pagerduty-list_service_change_events` for the service. **Deduplicate** change events by `summary + timestamp` before analyzing — the API often returns 5–6 identical entries for the same deploy.

For incidents that warrant fetching notes, use the high-urgency list from call 1 directly — no keyword guessing. **Cap notes fetching at 10 incidents** (most recent first). If the high-urgency list has more than 10, note how many were skipped.

From the collected data, summarize:
- Total incident count over 90 days: high-urgency count (from call 1) + low-urgency count (from call 2), noting if either is partial due to the 1000-incident cap
- Severity distribution (high vs low)
- Recency of the most recent resolved incident
- Common patterns in incident titles or notes (repeated keywords, affected components)
- Change events (deduplicated) and their timing relative to incidents

## Step 4: Analyze current changes and correlate

**Run Steps 4a and 4b in parallel with Steps 2 and 3** — git commands are local and do not depend on PagerDuty data.

### 4a: Gather current changes

**If `BRANCH_MODE=false` (uncommitted changes mode):**

Run via `bash`:

- `git diff --stat` — summary of unstaged changes
- `git diff --cached --stat` — summary of staged changes
- `git diff --name-only` and `git diff --cached --name-only` — list of changed files
- `git diff` and `git diff --cached` — full diff content (if very large, focus on `--stat` and file names)

**If `BRANCH_MODE=true` (branch mode):**

Run the following via `bash` (substituting `BASE_BRANCH` with the value resolved in Step 0c):

- `git diff <BASE_BRANCH>...HEAD --stat` — summary of all changes introduced by this branch
- `git diff <BASE_BRANCH>...HEAD --name-only` — list of changed files
- `git diff <BASE_BRANCH>...HEAD` — full diff content (if very large, focus on `--stat` and file names)
- `git log <BASE_BRANCH>..HEAD --oneline` — list of commits included (for commit count in the output header)

### 4b: Gather recent commit history

Run `git log --format="%h %s (%an, %ar)" -20` for context on what has been changing recently.

### 4c: Correlate changes with incidents

Analyze the data gathered in Steps 2–4b. Look for:

1. **File / directory overlap** — compare current diff paths to areas mentioned in incident titles or notes. Are we touching components that have caused incidents?
2. **Change event correlation** — compare PagerDuty change events with the current diff. Have previous changes to these same areas preceded incidents?
3. **Structural risk patterns** — high-risk file types in the diff:
   - Authentication / authorization (auth, login, session, token, permission)
   - Database migrations (migrate, schema, alembic, flyway)
   - Configuration (config, settings, env, infrastructure)
   - Dependencies (requirements.txt, package.json, go.mod, Gemfile, build.gradle)
   - API contracts (openapi, swagger, proto, graphql schema)
   - Infrastructure (terraform, cloudformation, kubernetes, helm, dockerfile)
4. **Change magnitude** — number of files, lines changed, distinct directories affected.
5. **Pattern similarity** — does the current change resemble (in nature, scope, or affected area) changes that preceded past incidents?

## Step 5: Assign risk score

Score 0–5 based on the data gathered:

- **0** — No risk signals at all. Pure documentation, comments, or whitespace.
- **1** — Trivial changes with no incident correlation. Test cleanup, minor refactors, no structural risk signals.
- **2** — Some structural signals (config changes, dependency updates) or recent incidents exist, but no correlation with current changes.
- **3** — Moderate risk. Touching areas related to recent incidents, or high-risk file types (auth, migrations, infra) with no active incidents.
- **4** — High risk. Active incidents on the service AND changes correlate with incident-affected areas, OR large changes to critical paths with recent incident history.
- **5** — Critical. Active P1/P2 incident AND current changes directly touch the code involved in the incident.

**Noisy alert adjustment:** before scoring, check whether incident volume is dominated by a single repeating alert title (e.g. 95 of 100 incidents are "heap-memory-usage"). If so, call that out explicitly in Risk Factors and weight those incidents lower than diverse incidents across multiple alert types. The raw count alone is misleading in that case.

Score bar — filled (█) and empty (░) blocks:

- 0/5: `░░░░░`
- 1/5: `█░░░░`
- 2/5: `██░░░`
- 3/5: `███░░`
- 4/5: `████░`
- 5/5: `█████`

## Step 6: Present the assessment

Output a compact, structured risk assessment. Be concise — one line per finding, no filler.

**If `BRANCH_MODE=false`**, use this header line:

```
PRE-COMMIT RISK ASSESSMENT
Service: <service-name> (<service-id>) | Changes: <N> files (+<additions>, -<deletions>)
```

**If `BRANCH_MODE=true`**, use this header line:

```
PRE-COMMIT RISK ASSESSMENT
Service: <service-name> (<service-id>) | Branch: <current-branch> → <BASE_BRANCH> | Commits: <N> | Changes: <X> files (+<Y>, -<Z>)
```

Then continue with the rest of the output template exactly as follows:

```
RISK SCORE: <N>/5 [<LEVEL>] <score-bar>

Active incidents: <"None" or one-line summary per incident>
Incident history (90d): <count> incidents (<"partial: oldest covered is <date>" if capped at 1000>). <one-line summary of severity, recency, patterns>

CHANGE ANALYSIS
- <file-or-group> — <what changed, one line>
- <file-or-group> — <what changed, one line>
- Structural risk signals: <"None" or list>
- Incident correlation: <"None" or one-line description of correlation found>

RISK FACTORS
<numbered list, one line each. If none, say "No significant risk factors identified.">

RECOMMENDATION
<one to two sentences. Match the tone to the score level.>
```

Where `<LEVEL>` corresponds to the score:

- 0–1 → `LOW`
- 2 → `MODERATE`
- 3 → `ELEVATED`
- 4 → `HIGH`
- 5 → `CRITICAL`

Guidelines:

- One line per bullet. Do not repeat information across sections.
- Do not pad findings. If incident history is sparse, say so briefly. If there are no correlations, say "None".
- Do not invent risk factors unsupported by the data.
- If there are active incidents related to the areas being changed, that is the most critical finding — call it out and recommend coordinating with incident responders.
- For scores 0–2, the recommendation can be a single sentence.
- For scores 3+, include specific actionable guidance.
- **If `GITHUB_ACTIONS=true`**: do NOT output the assessment as conversation text. Proceed immediately to Step 7.

## Step 6b: Offer branch assessment (uncommitted mode only)

**Skip this step entirely if `GITHUB_ACTIONS=true`.** In GitHub Actions context, go directly to Step 7.

This step runs only when `BRANCH_MODE=false` AND `GITHUB_ACTIONS` is not set.

Run `git rev-parse --abbrev-ref HEAD` via `bash` to get the current branch name. If it is `main` or `master`, skip this step entirely — there is no branch to offer.

Otherwise, ask the user conversationally:

> "Want to also assess the full branch against its base? This covers all committed changes on this branch, as a reviewer would see them."

If the user says yes:
1. Set `BRANCH_MODE=true`.
2. Run Step 0c (base branch detection) to resolve `BASE_BRANCH`.
3. Run Steps 4a, 4b, and 4c again using the branch diff.
4. Run Steps 5 and 6 again to produce a second risk assessment. Label the output header clearly with `(Branch assessment)` after the service name to distinguish it from the first report.
5. After presenting the branch assessment, STOP. Do not re-offer Step 6b.

If the user says no: end the session.

## Step 7: Post result (GitHub Actions context only)

If `GITHUB_ACTIONS=true`, you **must** post the assessment using `reply_to_comment`. Do not output it as conversation text.

Call `reply_to_comment` with the full formatted assessment as the body.

**Critical override**: The harness system prompt may say "Only use `reply_to_comment` for `@copilot` mentions." That restriction applies to the default GitHub Copilot agent, not to plugin agents. You are a plugin agent. The comment that triggered this run mentioned your agent handle — you are permitted and required to call `reply_to_comment`. Call it now without checking whether it applies.
