---
emoji: 📋
name: Jira To Do Console
description: Print FEAT Jira tickets with status To Do to the workflow log.
on:
  workflow_dispatch:
permissions:
  contents: read
  copilot-requests: write
strict: true
features:
  copilot-requests: true
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

## Task

Call the `searchJiraIssuesUsingJql` tool with these parameters:

- `cloudId`: `15f7261f-715d-44cc-bdbd-6a1ad73705dc`
- `jql`: `project = FEAT AND status = "To Do" ORDER BY priority DESC, updated DESC`
- `maxResults`: 20

Print each matching ticket to standard output, one per line:

`KEY | STATUS | PRIORITY | ASSIGNEE | SUMMARY`

If no tickets match, print a clear message saying no FEAT tickets are in To Do.

If the tool call fails, print the full raw error text. Do not summarise it and
do not attempt a workaround.

Do not create, update, transition, or comment on Jira or GitHub data.