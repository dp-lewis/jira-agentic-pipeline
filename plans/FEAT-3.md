# FEAT-3: Document the Jira-to-implementation agent workflow

Jira: https://dplewisdev.atlassian.net/browse/FEAT-3
Planned: 2026-08-15T01:44:56Z · Plan v1

## Scope and acceptance criteria

Add a single new documentation file, `docs/jira-agent-workflow.md`, describing
the end-to-end automated workflow that takes a Jira ticket from planning
through implementation and review. No workflow behavior changes.

Acceptance criteria (from the ticket):

1. `docs/jira-agent-workflow.md` exists and uses clear Markdown headings.
2. It accurately describes the workflow states, labels, PR lifecycle, and
   Jira updates.
3. It includes a concise end-to-end flow diagram using Mermaid or ASCII text.
4. It identifies the required secrets (`ATLASSIAN_MCP_BASIC`,
   `GH_AW_CI_TRIGGER_TOKEN`) without exposing their values.
5. It explains that PRs are not merged automatically.
6. No workflow behavior is changed (this is documentation only).

## Repository areas

The repository (`dp-lewis/automation-tests`) currently contains only
`README.md` and `.vscode/settings.json` — there is no existing `docs/`
directory, no committed `.github/workflows/` automation definitions, and no
prior documentation conventions to follow. This plan therefore:

- Creates a new `docs/` directory containing only `docs/jira-agent-workflow.md`.
- Does not reference any specific existing workflow YAML files, since none are
  present in the checked-out repository at planning time. The content should
  be written generically based on the process described in the ticket body
  (ticket intake → plan review → implementation approval → implementation
  review → operational guidance), rather than by inspecting workflow
  definitions that could not be located.

## Implementation steps

1. Create the `docs/` directory (if not already present) and the file
   `docs/jira-agent-workflow.md`.
2. Add a top-level heading and short intro paragraph summarizing the purpose
   of the automated Jira-to-implementation workflow.
3. Add a "Ticket intake" section describing:
   - Eligibility: ticket must be in `To Do` status and labeled `agent-ready`.
   - The planner selects exactly one eligible ticket and opens a draft PR.
4. Add a "Plan review" section describing:
   - The draft PR contains `plans/<TICKET-KEY>.md`.
   - Jira receives a "plan ready for review" comment containing the PR link.
   - The Jira label transitions from `agent-ready` to `agent-planned`.
5. Add an "Implementation approval" section describing:
   - A reviewer adds the `plan-approved` label to the draft plan PR.
   - The implementation workflow applies the approved plan on the same PR
     branch.
6. Add an "Implementation review" section describing:
   - The workflow runs applicable validation (tests/build/lint as defined by
     the target repo).
   - Passing implementations move the PR out of draft status (ready for
     human review).
   - Jira receives an "implementation ready for review" comment with the PR
     link, changed areas, and validation results.
   - Blocked or failed-validation work remains in draft status and does not
     update Jira.
7. Add an "Operational guidance" section describing:
   - Required GitHub repository secrets: `ATLASSIAN_MCP_BASIC` and
     `GH_AW_CI_TRIGGER_TOKEN` (name only, no values).
   - Recovery guidance for a blocked plan or blocked implementation (e.g.,
     add missing details/decisions to the Jira ticket or PR discussion and
     re-trigger the relevant workflow).
   - A short list of the workflow names and their responsibilities (planner
     workflow, implementation workflow, post-PR handoff workflow).
8. Add a concise end-to-end flow diagram in Mermaid (`graph TD` or
   `stateDiagram-v2`) or, as a fallback if Mermaid rendering cannot be
   verified, an ASCII flow, showing: `To Do + agent-ready` → draft plan PR
   opened → `agent-planned` → `plan-approved` label → implementation applied
   → validation → (ready-for-review PR + Jira comment) or (blocked/draft, no
   Jira update).
9. Explicitly state in the document that pull requests are never merged
   automatically by the automation — merging remains a manual, human step.

## Validation

- No test suite or build tooling exists in this repository at planning time
  (no `package.json`, CI config, or docs linter was found). Validation is
  limited to:
  - Manually reviewing that `docs/jira-agent-workflow.md` renders correctly
    as Markdown (headings, lists, and the Mermaid/ASCII diagram block).
  - Confirming all six acceptance criteria are explicitly satisfied by
    re-reading the finished document against the ticket's acceptance
    criteria list.
- If a Markdown linter or docs-build step is introduced before this ticket is
  implemented, run it against the new file.

## Risks, dependencies, and open questions

- The ticket describes workflow states, labels, and secrets that are not
  currently visible anywhere in this checked-out repository (no
  `.github/workflows/` directory was found). The plan assumes the ticket
  description is the authoritative source for these details; the
  implementer should confirm no separate/private workflow definitions exist
  that should instead be treated as ground truth.
- Open question: should the diagram be Mermaid or ASCII? The ticket allows
  either; recommend Mermaid for GitHub's native rendering support, with
  ASCII only as a fallback if Mermaid support in the target rendering
  context is uncertain.
- No linked epics, parent issues, or other referenced Jira keys were noted in
  the ticket description that require follow-up.
