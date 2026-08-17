# FEAT-3: Document the Jira-to-implementation agent workflow

Ticket: FEAT-3
Planned: 2026-08-17T05:04:59Z · Plan v1

## Scope and acceptance criteria

Add a new documentation file, `docs/jira-agent-workflow.md`, describing the
automated pipeline that takes a Jira ticket from planning through
implementation and review. This is a documentation-only change: no workflow
`.md`/`.lock.yml` files, actions, or agent instructions are modified.

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
- Reference material to draw from (read-only, not modified):
  - `.github/workflows/jira-plan-create.md` — ticket intake, preflight gate,
    plan file creation, draft PR creation.
  - `.github/workflows/jira-plan-notify.md` — validates the draft plan PR,
    comments "Plan ready for review" on Jira, moves `agent-ready` →
    `agent-planned`.
  - `.github/workflows/plan-implement.md` — triggered by the `plan-approved`
    label; implements the plan on the same PR branch, pushes changes, updates
    PR title/body, marks the PR ready for review, posts the
    `## Implementation complete` PR comment. Never merges.
  - `.github/workflows/jira-implement-notify.md` — triggered after
    `plan-implement` succeeds; verifies the completion comment and PR state,
    then comments "Implementation ready for review" on Jira. Never changes
    Jira status/labels beyond confirming `agent-planned`.
  - `.github/workflows/plan-comment-respond.md` — handles reviewer comments
    on an approved plan PR (human reviewers unlimited; the
    `copilot-pull-request-reviewer[bot]` capped once per PR via the
    `copilot-review-addressed` label).
  - `.github/workflows/jira-ticket-release.md` — triggered when a plan PR is
    closed; on merge, comments "Implementation delivered" and drops
    `agent-planned`; on close-without-merge, comments "Plan withdrawn" and
    replaces `agent-planned` with the terminal `agent-rejected` label (only a
    human can move a ticket back to `agent-ready`).
  - `.github/workflows/jira-preflight.md` — manual `workflow_dispatch` check
    that verifies configuration variables, the `ATLASSIAN_MCP_BASIC` and
    `GH_AW_CI_TRIGGER_TOKEN` secrets (presence only, never their values),
    required labels, and live Jira connectivity; opens a single "Blocking" or
    "Degraded" issue when unhealthy, or a no-op summary when "Ready".
  - `AGENTS.md` — repository conventions, out-of-bounds paths, and the
    project's own note that there is no automated validation.

## Implementation steps

1. Create `docs/jira-agent-workflow.md` with headings covering:
   - Overview / purpose of the pipeline.
   - Ticket intake: `To Do` + `agent-ready` label required; the planner
     (`jira-plan-create`) selects one eligible ticket via JQL scoped to the
     configured project keys, runs a preflight actionability gate, and opens
     a **draft** pull request containing `plans/<TICKET-KEY>.md` on branch
     `plan/<TICKET-KEY>`.
   - Plan review: `jira-plan-notify` validates the draft PR (title, branch,
     single changed plan file), posts a Jira comment with the PR link
     (idempotent via an HTML-comment marker), and swaps `agent-ready` for
     `agent-planned`.
   - Implementation approval: a human reviewer adds `plan-approved` to the
     draft PR. This label is the sole trigger for `plan-implement`.
   - Implementation: `plan-implement` implements the plan on the same branch,
     runs whatever validation `AGENTS.md` declares (or reports none exists),
     pushes via a safe output, updates the PR title/body, marks the PR ready
     for review (still not merged), and posts the `## Implementation
     complete` comment — or posts an `## Implementation blocked` comment and
     stops if it cannot proceed.
   - Implementation review notification: `jira-implement-notify` confirms the
     completion comment and non-draft state, then comments "Implementation
     ready for review" on Jira with changed areas and validation results.
     Blocked/failed work never reaches this step, so Jira is not touched.
   - Reviewer follow-up: `plan-comment-respond` lets a human reviewer (or a
     single capped pass from the Copilot PR reviewer bot, gated by the
     `copilot-review-addressed` label) request further changes on the same
     branch without re-triggering planning.
   - Ticket release: `jira-ticket-release` fires when the plan PR closes.
     Merged → "Implementation delivered" comment, ticket leaves the pipeline
     (label removed, no replacement). Closed without merging → "Plan
     withdrawn" comment and the terminal `agent-rejected` label; only a human
     can move it back to `agent-ready`.
   - Operational guidance: required secrets `ATLASSIAN_MCP_BASIC` (Atlassian
     MCP auth) and `GH_AW_CI_TRIGGER_TOKEN` (used to trigger notify/respond
     follow-ups); state only that they must be configured, never a value or
     format. Note `jira-preflight.md` as the `workflow_dispatch` health check
     that reports Blocking/Degraded/Ready and how to recover (e.g. removing
     `agent-rejected` and adding `agent-ready` to resume a withdrawn ticket).
   - List workflow names and one-line responsibilities for: `Jira: Preflight`,
     `Jira: Create Plan` (`jira-plan-create`), `Jira: Notify Plan Ready`,
     `Plan: Implement`, `Jira: Notify Implementation Ready`,
     `Plan: Respond to Comments`, `Jira: Release Ticket`.
   - Explicitly state that no pull request in this pipeline is ever merged
     automatically; merging is always a human action.
2. Include a Mermaid flowchart (or ASCII fallback) showing the state
   transitions: `agent-ready` → (plan PR opened, draft) → `agent-planned` →
   (human adds `plan-approved`) → implementation → PR marked ready for review
   → human merges or closes → `agent-planned` removed / `agent-rejected`.
3. Cross-check every label, secret name, and workflow name mentioned in the
   doc against the actual frontmatter/body in `.github/workflows/*.md` to
   avoid drift.
4. Do not touch `plans/**`, `.github/workflows/**`, `AGENTS.md`, or any other
   excluded path.

## Validation

No automated validation is declared or discoverable in this repository.

Per `AGENTS.md`, verify by inspection: confirm the new file matches this
approved plan, that its Markdown renders correctly (headings, code fences,
and any Mermaid block properly closed), and that internal links resolve.

## Risks, dependencies, and open questions

- The ticket description does not specify a preferred diagram style (Mermaid
  vs. ASCII); this plan defaults to Mermaid with an ASCII option only if
  Mermaid rendering is a concern for the target viewer — no functional risk
  either way.
- No other ticket keys were referenced in FEAT-3 requiring follow-up.
- The doc must stay in sync with workflow files as they evolve; since
  `.github/workflows/**` is out of bounds for the implementer, future
  workflow changes may cause this doc to drift and require a separate
  documentation-update ticket.
