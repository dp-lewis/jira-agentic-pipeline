# FEAT-3: Document the Jira-to-implementation agent workflow

Jira: https://dplewisdev.atlassian.net/browse/FEAT-3
Planned: 2026-08-16T04:50:57Z · Plan v1

## Scope and acceptance criteria

Add a new documentation file, `docs/jira-agent-workflow.md`, describing the
end-to-end automated pipeline that takes a Jira ticket from planning through
implementation and review. This is a documentation-only change: no workflow
behavior may be modified.

Acceptance criteria (from the ticket):

1. `docs/jira-agent-workflow.md` exists and uses clear Markdown headings.
2. It accurately describes the workflow states, labels, PR lifecycle, and
   Jira updates.
3. It includes a concise end-to-end flow diagram using Mermaid or ASCII text.
4. It identifies the required secrets without exposing their values.
5. It explains that PRs are not merged automatically.
6. No workflow behavior is changed.

## Repository areas

- New file: `docs/jira-agent-workflow.md` (only file to add/change).
- Reference material (read-only, not modified) for grounding the
  documentation in the actual implementation:
  - `.github/workflows/jira-todo-ticket-planner.md` — selects one
    `agent-ready` ticket in `To Do`, writes `plans/<TICKET-KEY>.md`, opens a
    draft PR on branch `plan/<TICKET-KEY>`.
  - `.github/workflows/jira-plan-pr-handoff.md` — on plan PR `synchronize`,
    comments the PR link back to Jira and (per the pipeline design) moves the
    ticket from `agent-ready` to `agent-planned`.
  - `.github/workflows/jira-plan-pr-closed.md` — on plan PR `closed`,
    releases the ticket back into the pipeline.
  - `.github/workflows/implement-approved-jira-plan.md` — triggered by the
    `plan-approved` label on the draft PR; applies the plan on the same PR
    branch, guarded by `required-labels: [plan-approved]` and an
    `excluded-files` list that protects `.github/workflows/**`,
    `.github/actions/**`, `.github/aw/**`, `.github/skills/**`, and
    `.github/copilot-instructions.md`.
  - `.github/workflows/jira-implementation-pr-handoff.md` — runs after a
    successful implementation workflow run, comments Jira with an
    "implementation ready for review" update and PR link; only fires on
    `conclusion == 'success'`.
  - `.github/workflows/respond-to-pr-comment.md` — handles reviewer follow-up
    comments on approved plan PRs, gated to `plan-approved` labeled PRs and
    trusted commenters.
  - `.github/workflows/jira-pipeline-preflight.md` — deterministic
    configuration/secret checks (`ATLASSIAN_MCP_BASIC`,
    `GH_AW_CI_TRIGGER_TOKEN`, and the `vars.JIRA_*` / `vars.PLAN_MAX_FILES`
    configuration variables).
  - `AGENTS.md` — repository conventions (labels, branch/plan-file naming,
    excluded paths, validation policy) to cross-check documentation accuracy.
  - `docs/portability-plan.md` — existing docs file, for house style
    reference only.

## Implementation steps

1. Read each workflow file listed above in full (frontmatter + prompt body)
   to confirm exact trigger events, guard conditions, label transitions, and
   secret names, so the documentation matches the compiled behavior rather
   than assumption.
2. Create `docs/jira-agent-workflow.md` with sections covering:
   - Overview / purpose of the pipeline.
   - Ticket intake: `To Do` + `agent-ready` label requirement, planner
     selection logic (JQL scoped by configured project keys).
   - Plan review: draft PR containing `plans/<TICKET-KEY>.md`, Jira comment
     with PR link, label transition `agent-ready` → `agent-planned`.
   - Implementation approval: human applies `plan-approved` to the draft PR;
     no auto-merge at any stage.
   - Implementation: the implementation workflow updates the same PR branch,
     respecting excluded paths; runs whatever validation the target repo
     declares (or none, if undeclared).
   - Implementation review: passing runs move the PR out of draft and post a
     Jira "ready for review" comment with the PR link, changed areas, and
     validation results; failed/blocked work stays draft and Jira is not
     updated.
   - Operational guidance: required secrets `ATLASSIAN_MCP_BASIC` and
     `GH_AW_CI_TRIGGER_TOKEN` (names only, no values), how to recover a
     blocked plan or implementation (e.g., closing the PR to release the
     ticket, or removing `agent-rejected` manually per `AGENTS.md`), and a
     short table of workflow names and responsibilities.
   - A Mermaid flowchart (or ASCII fallback) summarizing: Jira ticket →
     planner → draft plan PR → human approval label → implementation →
     validation → ready-for-review Jira comment, explicitly noting merges are
     manual.
3. Proofread for Markdown correctness (headings render, code fences and
   Mermaid blocks are closed, internal links resolve) before finalizing.

## Validation

No automated validation is declared or discoverable in this repository.

## Risks, dependencies, and open questions

- The ticket does not specify whether the Mermaid diagram must render in
  GitHub's default Markdown preview only, or also in other renderers; a
  standard Mermaid `flowchart` block is assumed to satisfy this.
- Exact wording of Jira comments (e.g., "plan ready for review",
  "implementation ready for review") should be verified against the live
  prompt bodies of `jira-plan-pr-handoff.md` and
  `jira-implementation-pr-handoff.md` at implementation time, since prompt
  wording may differ slightly from the ticket's paraphrase.
- No linked issues or parent epics were read, per the planning workflow's
  scope restriction; if FEAT-3 references other tickets, none were found in
  the fields read (summary, description, status, labels, priority).
