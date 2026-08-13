---
emoji: 📋
name: Jira To Do Console
description: Print FEAT Jira tickets with status To Do to the workflow log.
on:
  workflow_dispatch:
permissions:
  copilot-requests: write
strict: true
features:
  copilot-requests: true
network:
  allowed:
    - defaults
    - mcp.atlassian.com
tools:
  bash: [atlassian]
  cli-proxy: true
mcp-servers:
  atlassian:
    type: http
    url: https://mcp.atlassian.com/v1/mcp
    headers:
      Authorization: Bearer ${{ secrets.ATLASSIAN_MCP_TOKEN }}
---

# Jira To Do Console

## Task

Use the Atlassian CLI to query Jira with this JQL:

```text
project = FEAT AND status = "To Do" ORDER BY priority DESC, updated DESC
```

First run `atlassian --help` to identify the available Jira search command,
then invoke that command. Keep its result visible in standard output so it
appears in the workflow's Actions log.

Request and display each matching ticket's key, summary, status, assignee, and
priority. If no tickets match, print a clear message stating that no `FEAT`
tickets are currently in `To Do`.

Do not create, update, transition, comment on, or otherwise modify Jira or
GitHub data.
