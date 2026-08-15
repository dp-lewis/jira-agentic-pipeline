---
emoji: 🩺
name: Jira Pipeline Preflight
description: Verify that this repository is correctly configured to run the Jira pipeline.
on:
  workflow_dispatch:
permissions:
  contents: read
  issues: read
  copilot-requests: write
strict: true
features:
  copilot-requests: true
imports:
  - shared/jira-pipeline-config.md
network:
  allowed:
    - defaults
    - mcp.atlassian.com
steps:
  - name: Run deterministic preflight checks
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      REPOSITORY: ${{ github.repository }}
      HAS_ATLASSIAN_SECRET: ${{ secrets.ATLASSIAN_MCP_BASIC != '' }}
      HAS_TRIGGER_TOKEN: ${{ secrets.GH_AW_CI_TRIGGER_TOKEN != '' }}
      VAR_JIRA_CLOUD_ID: ${{ vars.JIRA_CLOUD_ID }}
      VAR_JIRA_SITE_URL: ${{ vars.JIRA_SITE_URL }}
      VAR_JIRA_PROJECT_KEYS: ${{ vars.JIRA_PROJECT_KEYS }}
      VAR_JIRA_READY_STATUS: ${{ vars.JIRA_READY_STATUS }}
      VAR_PLAN_MAX_FILES: ${{ vars.PLAN_MAX_FILES }}
    run: |
      set -uo pipefail
      mkdir -p /tmp/gh-aw/agent
      results=/tmp/gh-aw/agent/preflight.json

      # present <value> -> "ok" when non-empty, "missing" otherwise
      present() { [ -n "${1:-}" ] && echo ok || echo missing; }

      # api_state <path> -> ok | missing | unknown:<reason>
      #
      # A 404 means the resource genuinely does not exist. Anything else —
      # notably a 403 from an under-permissioned token — means the check could
      # not run. Those must never be reported as "missing": a false "missing"
      # sends someone off to recreate a label that is already there.
      api_state() {
        local out rc
        out="$(gh api "$1" 2>&1)"
        rc=$?
        if [ $rc -eq 0 ]; then
          echo ok
        elif echo "$out" | grep -q "HTTP 404"; then
          echo missing
        else
          echo "unknown:$(echo "$out" | tr '\n' ' ' | cut -c1-80)"
        fi
      }

      # There is no Actions permission scope that lets GITHUB_TOKEN read the
      # repository's workflow-permissions setting, so this is expected to be
      # unknown on most runs. It is reported, not treated as a failure.
      # gh writes the error body to stdout before exiting non-zero, so guard
      # the assignment rather than using `|| echo` — otherwise the raw 403 JSON
      # is concatenated onto the fallback string.
      if pr_setting="$(gh api "repos/$REPOSITORY/actions/permissions/workflow" \
           --jq '.can_approve_pull_request_reviews' 2>/dev/null)" \
         && [ -n "$pr_setting" ]; then
        can_create_prs="$pr_setting"
      else
        can_create_prs="unknown:GITHUB_TOKEN cannot read repository administration settings; verify manually in Settings > Actions > General"
      fi

      jq -n \
        --arg cloud_id      "$(present "$VAR_JIRA_CLOUD_ID")" \
        --arg site_url      "$(present "$VAR_JIRA_SITE_URL")" \
        --arg project_keys  "$(present "$VAR_JIRA_PROJECT_KEYS")" \
        --arg ready_status  "${VAR_JIRA_READY_STATUS:-To Do (default)}" \
        --arg max_files     "${VAR_PLAN_MAX_FILES:-5 (default)}" \
        --arg atlassian     "$([ "$HAS_ATLASSIAN_SECRET" = "true" ] && echo ok || echo missing)" \
        --arg trigger       "$([ "$HAS_TRIGGER_TOKEN" = "true" ] && echo ok || echo missing)" \
        --arg plan_label    "$(api_state "repos/$REPOSITORY/labels/plan-approved")" \
        --arg review_label  "$(api_state "repos/$REPOSITORY/labels/copilot-review-addressed")" \
        --arg agents_md     "$(api_state "repos/$REPOSITORY/contents/AGENTS.md")" \
        --arg create_prs    "$can_create_prs" \
        '{
          variables: {
            JIRA_CLOUD_ID: $cloud_id,
            JIRA_SITE_URL: $site_url,
            JIRA_PROJECT_KEYS: $project_keys
          },
          effective: {
            JIRA_READY_STATUS: $ready_status,
            PLAN_MAX_FILES: $max_files
          },
          secrets: {
            ATLASSIAN_MCP_BASIC: $atlassian,
            GH_AW_CI_TRIGGER_TOKEN: $trigger
          },
          labels: {
            "plan-approved": $plan_label,
            "copilot-review-addressed": $review_label
          },
          repository: {
            "AGENTS.md": $agents_md,
            actions_can_create_pull_requests: $create_prs
          }
        }' > "$results"

      {
        echo "## Jira pipeline preflight"
        echo
        echo "| Check | Result |"
        echo "| --- | --- |"
        jq -r '
          [ (.variables   | to_entries[] | "Variable `\(.key)`|\(.value)"),
            (.effective   | to_entries[] | "Effective `\(.key)`|\(.value)"),
            (.secrets     | to_entries[] | "Secret `\(.key)`|\(.value)"),
            (.labels      | to_entries[] | "Label `\(.key)`|\(.value)"),
            (.repository  | to_entries[] | "Repository \(.key)|\(.value)")
          ] | .[] | "| " + (. | sub("\\|"; " | ")) + " |"
        ' "$results"
        echo
        echo "Live Jira connectivity is checked by the agent step below."
      } >> "$GITHUB_STEP_SUMMARY"

      cat "$results"
mcp-servers:
  atlassian:
    type: http
    url: https://mcp.atlassian.com/v1/mcp
    headers:
      Authorization: "Basic ${{ secrets.ATLASSIAN_MCP_BASIC }}"
    allowed:
      - "searchJiraIssuesUsingJql"
safe-outputs:
  create-issue:
    max: 1
  report-failure-as-issue: false
---

# Jira Pipeline Preflight

Verify that this repository is configured to run the Jira pipeline, and report
what is missing. Change nothing.

## Configuration

- Jira cloud ID: `${{ env.JIRA_CLOUD_ID }}`
- Jira project keys: `${{ env.JIRA_PROJECT_KEYS }}`
- Ready status: `${{ env.JIRA_READY_STATUS }}`

## 1. Read the deterministic results

Read `/tmp/gh-aw/agent/preflight.json`. It was produced before this step by a
shell check and is authoritative for variables, secrets, labels, and
repository settings. Do not re-derive those values or contradict them.

Every check holds one of three states, and the third is not a failure:

- `ok` — verified present or enabled.
- `missing` — verified absent. The API returned 404.
- `unknown:<reason>` — **the check could not run**, usually because the token
  lacked permission. This is not evidence of absence. Never report an
  `unknown` check as missing, never propose a fix for it as though the
  resource were absent, and never classify the repository as blocking because
  of one. List these separately as "could not be checked", quote the reason,
  and say what would let the check run.

## 2. Check live Jira connectivity

Only if all three variables in `/tmp/gh-aw/agent/preflight.json` are `ok` and
the `ATLASSIAN_MCP_BASIC` secret is `ok`, call `searchJiraIssuesUsingJql` with:

- `cloudId`: the configured Jira cloud ID
- `jql`: `project IN (<JIRA_PROJECT_KEYS>) AND status = "<JIRA_READY_STATUS>"
  ORDER BY priority DESC, updated DESC`
- `maxResults`: 5

Render `${{ env.JIRA_PROJECT_KEYS }}` as a JQL list, for example
`project IN (FEAT, PLAT)`. Never issue a search without the project filter.

Record one of three outcomes:

- **ok** — the call succeeded. Note how many tickets are ready to plan. Zero
  is a valid, healthy result.
- **auth failed** — the call was rejected. The secret is set but not valid for
  this Atlassian site.
- **not attempted** — a prerequisite above was missing.

If the call fails, capture the full raw error text. Do not summarise it and do
not attempt a workaround.

## 3. Report

Classify the configuration:

- **Blocking** — any required variable `missing`, the Atlassian secret
  `missing` or failing to authenticate, or `plan-approved` `missing`. The
  pipeline cannot run.
- **Degraded** — `GH_AW_CI_TRIGGER_TOKEN` missing (plan PRs will not notify
  Jira), `copilot-review-addressed` missing (the review responder cannot mark
  itself complete and may repeat), `actions_can_create_pull_requests` explicitly
  `false` (the planner cannot open its PR), or `AGENTS.md` missing (changes
  will be implemented without declared validation).
- **Ready** — everything above is satisfied.

Classify on `missing` and explicit `false` only. A check in an `unknown` state
does not raise the classification; report it under "could not be checked" and
carry on.

If the result is **Ready**, call `noop` with a one-line summary including the
count of ready tickets. Do not create an issue. A healthy repository must not
accumulate preflight issues.

Otherwise create exactly one issue titled
`Jira pipeline preflight: <blocking|degraded>` whose body contains:

```markdown
## Preflight result

<blocking or degraded, and the one-sentence consequence>

### Failing checks

| Check | Result | Fix |
| --- | --- | --- |

### Could not be checked

<omit this section entirely when no check is in an unknown state>

### Passing checks

<compact list>
```

In the `Fix` column give the exact command or setting, for example
`gh variable set JIRA_CLOUD_ID --body "<id>"` or
`gh label create copilot-review-addressed --color EDEDED`. Name the specific
remedy; never write "check configuration".

Do not modify Jira, labels, variables, secrets, or any repository content.
