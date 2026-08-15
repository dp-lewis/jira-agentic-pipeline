---
emoji: 📋
name: Jira To Do Console
description: Print ready Jira tickets in the configured projects to the workflow log.
on:
  workflow_dispatch:
permissions:
  contents: read
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
mcp-servers:
  atlassian:
    type: http
    url: https://mcp.atlassian.com/v1/mcp
    headers:
      Authorization: "Basic ${{ secrets.ATLASSIAN_MCP_BASIC }}"
    allowed:
      - "searchJiraIssuesUsingJql"
---

# Jira To Do Console

## Configuration

- Jira cloud ID: `${{ env.JIRA_CLOUD_ID }}`
- Jira project keys: `${{ env.JIRA_PROJECT_KEYS }}`
- Ready status: `${{ env.JIRA_READY_STATUS }}`

## Task

Apply the configuration gate above. Then call the `searchJiraIssuesUsingJql`
tool with these parameters:

- `cloudId`: the configured Jira cloud ID
- `jql`: `project IN (<JIRA_PROJECT_KEYS>) AND status = "<JIRA_READY_STATUS>"
  ORDER BY priority DESC, updated DESC`
- `maxResults`: 20

Render `${{ env.JIRA_PROJECT_KEYS }}` as a JQL list, for example
`project IN (FEAT, PLAT)`. Never issue a search without the project filter.

Print each matching ticket to standard output, one per line:

`KEY | STATUS | PRIORITY | ASSIGNEE | SUMMARY`

If no tickets match, print a clear message saying no tickets in the configured
projects are in the ready status.

If the tool call fails, print the full raw error text. Do not summarise it and
do not attempt a workaround.

Do not create, update, transition, or comment on Jira or GitHub data.