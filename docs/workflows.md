# Workflows

Two layers. The `plan-*` workflows are the engine and need no issue tracker.
The `jira-*` workflows bind that engine to Jira.

## `plan-*` — the engine

Driven entirely by a `plans/<KEY>.md` file and the `plan-approved` label. No
Atlassian connection, no repository variables, no tracker of any kind. Adopt
these two on their own and drive them with a hand-written plan, or wire them
to Linear or GitHub Issues instead.

| File | Fires on | Does |
| --- | --- | --- |
| `plan-implement.md` | `plan-approved` added to a draft `plan/*` PR | Implements the plan, retitles the PR, marks it ready |
| `plan-comment-respond.md` | Comment from a write-access user or Copilot | Answers questions on any plan PR; applies in-scope changes once implemented |

## `jira-*` — the Jira integration

These need `ATLASSIAN_MCP_BASIC` and the repository variables, and all import
`shared/jira-pipeline-config.md`.

| File | Fires on | Does |
| --- | --- | --- |
| `jira-plan-create.md` | Weekday schedule, or manual | Picks one eligible ticket and opens a draft plan PR |
| `jira-plan-notify.md` | Plan file pushed to a draft `plan/*` PR | Tells Jira the plan is ready; `agent-ready` → `agent-planned` |
| `jira-implement-notify.md` | `Plan: Implement` completing | Tells Jira the implementation needs review |
| `jira-ticket-release.md` | Plan PR closed or merged | Releases the ticket from the pipeline |
| `jira-preflight.md` | Manual | Checks configuration and reports what is missing |

## `shared/`

Not workflows. `jira-pipeline-config.md` carries the `env:` block that maps
repository variables into the Jira workflows, plus the policy prose they share.

## Things that will bite you

**Workflows cannot live in subdirectories.** GitHub only reads
`.github/workflows/*.yml` at the top level, and `gh aw compile` *silently
skips* any `.md` in a subdirectory — no error, no lockfile, the workflow just
stops existing. `shared/` works only because nothing in it is a workflow.

**`jira-implement-notify` references `Plan: Implement` by display name.**
Renaming that workflow's `name:` without updating the `workflow_run` trigger
breaks the Jira notification silently.

**Display names containing a colon must be quoted.** `name: Plan: Implement`
is invalid YAML.

**Always recompile and commit both files.** Each `.md` has a generated
`.lock.yml`; the lockfile is what actually runs.

```bash
gh aw compile --strict --approve
```

This file lives in `docs/` rather than beside the workflows on purpose. gh-aw
treats every `.md` under `.github/workflows/` as a workflow, so a README there
is picked up as a shared workflow and reported as an error during every
install — cosmetic, but a poor first impression for someone installing the
package.
