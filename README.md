# Jira → Pull Request Agentic Pipeline

A staged, human-governed pipeline that turns a well-specified Jira ticket into
a planned, implemented, and reviewed pull request. Agents do bounded work.
People decide what is ready to plan, approve the plan, review the result, and
merge it.

**Nothing merges automatically.** Implementation does not begin until a human
applies a label to a plan they have read.

Built on [GitHub Agentic Workflows](https://github.com/github/gh-aw) (`gh aw`)
and the Atlassian MCP server.

## How it works

```mermaid
flowchart TD
    A[Jira ticket: ready status + agent-ready] --> B[Planner]
    B --> C[Draft PR: plan/TICKET-KEY]
    C --> D[Jira comment: plan ready + agent-planned]
    D --> E{Human reviews plan}
    E -->|adds plan-approved| F[Implementation]
    E -->|closes PR| X[Jira: agent-rejected, released]
    F --> G[PR retitled, marked ready for review]
    G --> H[Jira comment: implementation ready]
    G --> I[Copilot review]
    I --> J[One bounded fix cycle]
    J --> K{Human reviews and merges}
    K -->|merged| L[Jira: delivered, agent-planned removed]
    K -->|closed| X
```

The ticket never leaves human control. The agent's authority is capped at
"open a draft PR containing one markdown file" until someone approves it.

### The six workflows

| Workflow | Trigger | What it may do |
| --- | --- | --- |
| `jira-todo-ticket-planner` | Daily on weekdays, or manual | Open a draft PR containing only `plans/<KEY>.md` |
| `jira-plan-pr-handoff` | Plan file pushed to a draft `plan/*` PR | Comment on Jira, swap `agent-ready` → `agent-planned` |
| `implement-approved-jira-plan` | Human adds `plan-approved` | Implement the plan, retitle the PR, mark ready for review |
| `jira-implementation-pr-handoff` | Implementation workflow succeeds | Comment on Jira that review is needed |
| `jira-plan-pr-closed` | Plan PR closed or merged | Release the ticket from the pipeline and record the outcome |
| `respond-to-pr-comment` | Any comment from a write-access user or Copilot | Answer a question, or apply a change that falls within the approved plan |

Each stage independently re-validates the PR — branch name, title, draft
state, labels, changed files — rather than trusting that the previous stage
behaved. A failed check is a `noop`, not a guess.

`jira-pipeline-preflight` verifies your configuration on demand — see
[Verify](#7-verify). It is the only workflow here that is not part of the flow
itself.

## Requirements

- A repository using `gh aw` ([install](https://github.com/github/gh-aw))
- Jira Cloud, with an account that can read issues, comment, and edit labels
- GitHub Actions enabled, with permission to create pull requests
- Optional: a repository ruleset requesting automatic Copilot review

## Install

### 1. Add the workflows

This repository is not yet published as a `gh aw` package. For now, copy
`.github/workflows/*.md`, `.github/workflows/shared/`, and compile:

```bash
gh aw compile --strict --approve
```

### 2. Declare your conventions

Copy [`templates/AGENTS.md`](templates/AGENTS.md) to your repository root and
fill it in — the `## Validation` section above all. This is what stops the
pipeline handing you an unverified diff.

### 3. Set repository variables

```bash
gh variable set JIRA_CLOUD_ID     --body "<your-atlassian-cloud-id>"
gh variable set JIRA_SITE_URL     --body "https://<your-site>.atlassian.net"
gh variable set JIRA_PROJECT_KEYS --body "PROJ"
```

The three above are required and have **no defaults on purpose**. If any is
unset, every workflow stops with a `noop` naming the missing variable rather
than running an unscoped Jira search. See
[Configuration](#configuration) for the optional ones.

### 4. Set secrets

```bash
gh secret set ATLASSIAN_MCP_BASIC       # Basic auth for the Atlassian MCP server
gh secret set GH_AW_CI_TRIGGER_TOKEN    # Fine-grained PAT, Contents read/write
```

`GH_AW_CI_TRIGGER_TOKEN` exists only to make an empty commit that triggers
downstream PR workflows, because pushes made with `GITHUB_TOKEN` do not fire
`pull_request` events. It grants no other authority.

### 5. Create the GitHub labels

```bash
gh label create plan-approved            --color 23C18F \
  --description "Human-approved plan; implementation may begin"
gh label create copilot-review-addressed --color EDEDED \
  --description "The one automatic Copilot response cycle has been used"
```

### 6. Allow Actions to create pull requests

Settings → Actions → General → **Allow GitHub Actions to create and approve
pull requests**. The PAT does not grant this.

### 7. Verify

```bash
gh aw run jira-pipeline-preflight
```

A read-only check of every step above: required variables, both secrets, the
two GitHub labels, the Actions pull-request setting, `AGENTS.md`, and live
Jira connectivity. Results appear in the run's job summary.

It classifies the repository as **ready**, **degraded** (the pipeline runs but
a stage is impaired), or **blocking** (it cannot run). A ready repository
produces no issue; anything else files exactly one, naming the precise command
to fix each failing check.

## The ticket contract

The planner will **refuse** a ticket, and comment on Jira explaining exactly
what is missing, when it:

- has no acceptance criteria, or criteria that cannot be tested
- does not identify which part of the system changes
- needs a product or design decision that has not been made
- is a bug report with no reproduction steps
- plainly needs more than `PLAN_MAX_FILES` files changed

Refusal is a feature. The planner will not invent requirements to make a
ticket pass. A ticket that gets refused is a ticket that would have produced a
bad PR.

To make a ticket eligible: set its status to your configured ready status
(default `To Do`) and add the Jira label `agent-ready`.

## Daily operation

1. Write a Jira ticket that meets the contract above. Label it `agent-ready`.
2. Wait for the planner, or run it manually:
   `gh aw run jira-todo-ticket-planner --raw-field ticket=PROJ-123`
3. **Read the plan PR.** This is the checkpoint that matters.
4. Add `plan-approved` to start implementation, or close the PR to stop.
5. Review the resulting implementation PR and merge it yourself.

### Reviewing by comment

Just comment normally on an implemented PR — no command or prefix. Write on the
conversation or inline on a diff line. The agent reads every comment from a
write-access user and decides what it is:

- **A change request** — it applies the change, validates, pushes, and replies.
- **A question** — it answers from evidence, citing the file or plan section,
  and pushes nothing.
- **Neither** — it stays silent. "LGTM", "merging Monday", and two humans
  talking to each other get no response. When genuinely unsure it stays quiet,
  because a missed request costs one follow-up comment while an unwanted reply
  on every remark makes the pipeline unpleasant to work with.

Change requests are bounded by the approved plan. One that would deliver
something the plan does not describe is refused with the specific gap named,
not quietly implemented. That keeps the plan as the contract: if you want
something genuinely new, update the ticket and let the planner re-plan it.

Two authors reach it, and only two: humans with write access, and
`copilot-pull-request-reviewer[bot]`. Everything else — including this
workflow's own replies — is rejected before the agent starts, so it cannot loop
and unauthorised comments cost nothing.

Copilot's inline comments are handled the same way, with one difference: it is
capped at **one response cycle per PR**, tracked by the
`copilot-review-addressed` label. Copilot re-reviews after every push, so
without a cap a fix would prompt a review would prompt a fix, indefinitely.
Humans are uncapped, because a human stops on their own. Once the label is set,
further Copilot comments on that PR are ignored and its remaining findings are
yours to weigh.

### Label lifecycle

| Label | Where | Meaning |
| --- | --- | --- |
| `agent-ready` | Jira | Eligible for automated planning |
| `agent-planned` | Jira | A draft plan PR exists, awaiting approval |
| `agent-rejected` | Jira | A plan PR was closed unmerged; released from the pipeline |
| `plan-approved` | GitHub PR | A human approved the plan; implementation may begin |
| `copilot-review-addressed` | GitHub PR | The one automatic response cycle is used up |

Closing a plan PR without merging is a supported way to say no. The ticket is
released from the pipeline with `agent-rejected` and a Jira comment explaining
what happened. That state is terminal by design — nothing restores
`agent-ready` automatically, because the planner would regenerate the same
plan from the same unchanged ticket and you would have to close it again. Fix
whatever made the plan unsuitable, then swap the label back yourself.

Merging a plan PR removes `agent-planned` and comments that the work was
delivered. Jira status is never changed by any workflow; moving the ticket to
Done stays a human decision.

## Configuration

Instance values live in repository variables, never in workflow text, so
`gh aw update` can deliver fixes without clobbering your setup.

| Variable | Required | Default | Example |
| --- | --- | --- | --- |
| `JIRA_CLOUD_ID` | yes | — | `15f7261f-…` |
| `JIRA_SITE_URL` | yes | — | `https://acme.atlassian.net` |
| `JIRA_PROJECT_KEYS` | yes | — | `PROJ` or `PROJ,PLAT` |
| `JIRA_READY_STATUS` | no | `To Do` | `Selected for Development` |
| `PLAN_MAX_FILES` | no | `5` | `8` |

These are wired in `.github/workflows/shared/jira-pipeline-config.md`, which
every Jira-touching workflow imports.

Per-project code conventions — test commands, file layout, style — belong in
an `AGENTS.md` at your repository root. Copy [`templates/AGENTS.md`](templates/AGENTS.md)
and fill it in.

Its `## Validation` section is the most important part: it declares the exact
commands that prove a change works. The planner copies them into the plan, so
you see the validation commitment *before* you approve; the implementer runs
them and blocks on failure.

If you declare nothing, the pipeline falls back to discovering your
conventions — and if it finds nothing runnable, it says so verbatim rather
than implying a change was verified. You will never be shown a `### Validation`
section that describes work that did not happen.

### What is deliberately not configurable

Branch prefix (`plan/*`), plan path (`plans/**`), PR title format, and the
four lifecycle labels are fixed conventions. The labels in particular stay
literal because `required-labels` is the approval security boundary — keeping
it a literal means you can audit what gates implementation by reading the
compiled `.lock.yml`.

## Safety model

- Jira tickets, PRs, plans, and review comments are treated as untrusted task
  data, never as instructions. Prompt-injection attempts in a ticket do not
  widen the agent's scope.
- The planner can only create a draft PR with one file under `plans/`.
- Implementation and review workflows cannot touch workflow files, agent
  instructions, credentials, `.env` files, keys, or any path matching
  `*credential*` or `*secret*`.
- All repository writes go through `safe-outputs`, not write-capable tools.
- Implementation requires a human-applied label. There is no auto-merge.
- Comment responses are gated on write access via `author_association`,
  checked before the agent starts, and change requests are bounded by the
  approved plan — a comment cannot introduce work that never went through plan
  review.
- Jira notifications carry hidden idempotency markers, so retries do not spam
  the ticket.
- Copilot review gets exactly one fix cycle, preventing review/fix loops.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Every run `noop`s naming a variable | That repository variable is unset |
| Planner finds nothing | No ticket in the configured projects has both the ready status and `agent-ready` |
| Plan PR created but Jira silent | `GH_AW_CI_TRIGGER_TOKEN` missing or lacking Contents write |
| `plan-approved` does nothing | PR is not a draft, or its title is not `<KEY>: plan` |
| Ticket stuck in `agent-planned` | Its plan PR is still open — close or merge it to release the ticket |
| Ticket in `agent-rejected` not replanned | Terminal by design; remove the label and add `agent-ready` |

Inspect any run with `gh aw logs <workflow> -c 1 --artifacts agent`. The
rendered prompt at `aw-prompts/prompt.txt` shows exactly what the agent saw,
including resolved configuration values.

## Repository layout

```
AGENTS.md                          this repo's own conventions and validation
templates/AGENTS.md                copy to your repo root and fill in
.github/workflows/
  shared/jira-pipeline-config.md   configuration seam (env + shared policy)
  *.md                             workflow sources — edit these
  *.lock.yml                       compiled output — generated, commit alongside
.github/skills/agentic-workflows/  gh-aw authoring router skill
docs/                              design notes and portability plan
```

After changing any workflow, recompile and commit both files together:

```bash
gh aw compile <workflow-id> --strict --approve
```

## Documentation

- [Design and rationale](docs/jira-agentic-workflow-summary.md) — how the
  pipeline works and the engineering lessons behind it
- [Portability plan](docs/portability-plan.md) — the work to make this
  installable elsewhere, including known gaps

## Known gaps

Tracked in the [portability plan](docs/portability-plan.md):

- No plan revision path — plans are v1 only
- Implementation pushes may not trigger CI on the PR
- Not yet published as an installable `gh aw` package
