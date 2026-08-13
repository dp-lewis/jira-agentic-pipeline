---
emoji: 📊
name: Daily Repository Activity Report
description: Publish a daily issue summarizing recent repository activity.
on:
  schedule: daily
  workflow_dispatch:
permissions:
  contents: read
  issues: read
  pull-requests: read
  copilot-requests: write
strict: true
features:
  copilot-requests: true
tools:
  github:
    mode: gh-proxy
    toolsets: [default]
steps:
  - name: Prepare daily activity data
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      REPOSITORY: ${{ github.repository }}
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/agent

      window_end="$(date -u +'%Y-%m-%dT%H:%M:%SZ')"
      window_start="$(date -u -d '24 hours ago' +'%Y-%m-%dT%H:%M:%SZ')"

      gh api "repos/$REPOSITORY/commits?since=$window_start&until=$window_end&per_page=100" \
        > /tmp/gh-aw/agent/commits.json
      gh api "repos/$REPOSITORY/issues?state=all&sort=updated&direction=desc&since=$window_start&per_page=100" \
        > /tmp/gh-aw/agent/items.json
      gh api "repos/$REPOSITORY/releases?per_page=100" \
        > /tmp/gh-aw/agent/releases.json

      jq -n \
        --arg window_start "$window_start" \
        --arg window_end "$window_end" \
        --slurpfile commits /tmp/gh-aw/agent/commits.json \
        --slurpfile items /tmp/gh-aw/agent/items.json \
        --slurpfile releases /tmp/gh-aw/agent/releases.json \
        '{
          window: {
            start_utc: $window_start,
            end_utc: $window_end,
            description: "Last 24 full hours ending at workflow start (UTC)"
          },
          activity: {
            commits: {
              count: ($commits[0] | length),
              limited_to_100: (($commits[0] | length) == 100),
              items: ($commits[0] | map({
                sha: .sha[0:7],
                committed_at: .commit.author.date
              }))
            },
            pull_requests: {
              count: ($items[0] | map(select(.pull_request != null)) | length),
              limited_to_100: (($items[0] | length) == 100),
              items: ($items[0] | map(select(.pull_request != null) | {
                number,
                state,
                created_at,
                updated_at,
                closed_at
              }))
            },
            issues: {
              count: ($items[0] | map(select(.pull_request == null)) | length),
              limited_to_100: (($items[0] | length) == 100),
              items: ($items[0] | map(select(.pull_request == null) | {
                number,
                state,
                created_at,
                updated_at,
                closed_at
              }))
            },
            releases: {
              count: ($releases[0] | map(select(.published_at != null and .published_at >= $window_start and .published_at <= $window_end)) | length),
              limited_to_100: (($releases[0] | length) == 100),
              items: ($releases[0] | map(select(.published_at != null and .published_at >= $window_start and .published_at <= $window_end) | {
                tag_name,
                published_at,
                prerelease
              }))
            }
          }
        }' > /tmp/gh-aw/agent/daily-activity.json
safe-outputs:
  mentions: false
  allowed-github-references: []
  create-issue:
    title-prefix: "Daily Activity Report: "
    close-older-issues: true
    close-older-key: "daily-activity-report"
    deduplicate-by-title: true
    expires: 30
---

# Daily Repository Activity Report

## Task

Read `/tmp/gh-aw/agent/daily-activity.json`. It contains the complete, bounded
activity dataset for the last 24 full hours ending at workflow start in UTC.
Do not fetch additional GitHub data.

If every activity count is zero, call `noop` with the evaluated UTC window.
Otherwise, create exactly one issue titled with the report's UTC end date in
`YYYY-MM-DD` format. Its stable deduplication key is
`daily-activity:<UTC end date>`.

Write a factual GitHub-flavored Markdown report using this structure:

- `### Summary`: the UTC window and totals for commits, pull requests, issues,
  and releases.
- `### Activity`: concise grouped facts for commits, pull requests, issues,
  and releases. State when a category was limited to 100 items.
- Put item-by-item details in `<details>` sections, with at most 20 items per
  category.
- `### Context`: the deduplication key and triggering workflow run reference.

Use only the supplied fields. Do not infer intent, invent descriptions, or
include user-authored titles, bodies, or commit messages.

## Safe Outputs

- Use `create-issue` only for a non-empty report.
- Call `noop` with a short reason when the activity window is empty.
