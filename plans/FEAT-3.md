# FEAT-3: Document the Jira-to-implementation agent workflow

Jira: https://dplewisdev.atlassian.net/browse/FEAT-3
Planned: 2026-08-15T02:51:51Z · Plan v1

## Scope and acceptance criteria

Add `docs/jira-agent-workflow.md` documenting the end-to-end Jira-to-PR agent
workflow implemented by this repository's `.github/workflows/*.md` agentic
workflows. No workflow behavior changes — this is a documentation-only ticket.

Acceptance criteria (from the ticket):

1. `docs/jira-agent-workflow.md` exists and uses clear Markdown headings.
2. It accurately describes the workflow states, labels, PR lifecycle, and Jira
   updates.
3. It includes a concise end-to-end flow diagram using Mermaid or ASCII text.
4. It identifies the required secrets (`ATLASSIAN_MCP_BASIC`,
   `GH_AW_CI_TRIGGER_TOKEN`) without exposing their values.
5. It explains that PRs are not merged automatically.
6. No workflow behavior is changed.

## Repository areas

- New file: `docs/jira-agent-workflow.md` (does not currently exist; `docs/`
  directory does not exist yet and must be created).
- Source of truth to document (read-only, not modified):
  - `.github/workflows/jira-todo-ticket-planner.md` — selects one
    `agent-ready` + `To Do` ticket, writes `plans/<TICKET-KEY>.md`, opens a
    draft PR on branch `plan/<TICKET-KEY>`.
  - `.github/workflows/jira-plan-pr-handoff.md` — on the draft plan PR,
    comments on Jira with the PR link and moves the ticket from
    `agent-ready` to `agent-planned`. Uses `ATLASSIAN_MCP_BASIC`.
  - `.github/workflows/implement-approved-jira-plan.md` — triggered when a
    draft PR with the `plan-approved` label and a single `plans/**` file
    changes; applies the plan on the same branch.
  - `.github/workflows/jira-implementation-pr-handoff.md` — validates the PR
    (draft, `plan-approved` label, matches plan title, contains
    `plans/<TICKET-KEY>.md` plus other changed files, ticket labeled
    `agent-planned`) and comments on Jira that implementation is ready for
    review, using `ATLASSIAN_MCP_BASIC`.
  - `.github/workflows/respond-to-copilot-review.md` — handles review
    feedback on `plan-approved` PRs, requiring `GH_AW_CI_TRIGGER_TOKEN`.
  - `.github/workflows/jira-todo-console.md` — auxiliary console/reporting
    workflow using `ATLASSIAN_MCP_BASIC`.
  - `.github/workflows/daily-activity-report.md` — unrelated reporting
    workflow; likely out of scope for this doc but should be checked during
    implementation in case it references the same workflow states.

## Implementation steps

1. Create the `docs/` directory and add `docs/jira-agent-workflow.md`.
2. Write an overview section summarizing the goal: automating a Jira ticket
   from `To Do`/`agent-ready` through a reviewed, implemented pull request.
3. Document "Ticket intake": eligibility criteria (`To Do` status,
   `agent-ready` label), how `jira-todo-ticket-planner.md` selects a ticket,
   and creation of the draft PR containing `plans/<TICKET-KEY>.md` on branch
   `plan/<TICKET-KEY>`.
4. Document "Plan review": how `jira-plan-pr-handoff.md` posts the "plan
   ready for review" Jira comment with the PR link and transitions the label
   from `agent-ready` to `agent-planned`.
5. Document "Implementation approval": a human reviewer adds the
   `plan-approved` label to the draft PR, triggering
   `implement-approved-jira-plan.md` to apply the plan on the same branch.
6. Document "Implementation review": validation run by the implementation
   workflow, the PR leaving draft status on success, the "implementation
   ready for review" Jira comment (PR link, changed areas, validation
   results) posted by `jira-implementation-pr-handoff.md`, and that
   blocked/failed work stays in draft without updating Jira.
7. Document "Operational guidance": list the two required repository
   secrets (`ATLASSIAN_MCP_BASIC`, `GH_AW_CI_TRIGGER_TOKEN`) by name only
   (no values), briefly note their purpose (Jira API auth; triggering CI on
   agent-pushed commits), and describe recovery from a blocked plan or
   implementation (e.g., re-run the relevant workflow, or manually adjust
   labels/comments).
8. List each workflow file under `.github/workflows/` and its
   responsibility in one line each, for quick orientation.
9. Add a concise Mermaid flowchart (or ASCII fallback) showing the states:
   `To Do + agent-ready` → draft PR + `plans/<KEY>.md` → `agent-planned` →
   `plan-approved` label → implementation applied → validation → PR ready
   for review (or blocked/draft).
10. Explicitly state that pull requests are never merged automatically by
    any of these workflows — merging remains a manual, human action.

## Validation

This is a documentation-only change; no code or workflow files are modified.
Validation is manual review:

- Confirm `docs/jira-agent-workflow.md` renders correctly as Markdown
  (headings, Mermaid code block syntax).
- Cross-check each documented state, label, and secret name against the
  corresponding `.github/workflows/*.md` file to ensure accuracy.
- Confirm no `.github/workflows/*.md` or `.lock.yml` file is changed.

## Risks, dependencies, and open questions

- The repository has no `AGENTS.md`, so there are no additional
  documented conventions beyond the workflow markdown files themselves.
- Open question: should `docs/jira-agent-workflow.md` also mention
  `jira-todo-console.md` and `daily-activity-report.md`, which are adjacent
  but not part of the core intake→implementation→review chain? This plan
  includes a brief one-line mention of each for completeness, but the ticket
  does not explicitly require it.
- Open question: the exact wording/format of the "implementation ready for
  review" comment (changed areas, validation results) should be taken
  verbatim from `jira-implementation-pr-handoff.md` during implementation to
  avoid drift between the doc and actual workflow behavior.
- No other Jira tickets or epics were referenced in FEAT-3's description.
