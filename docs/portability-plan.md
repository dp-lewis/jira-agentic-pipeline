# Making the Jira Pipeline Portable

## Goal

Turn this repository from a working single-instance experiment into an
installable package that another team can add to their own repository and
operate without editing prompt text.

The target install experience is: set a handful of repository variables and
two secrets, run a preflight workflow that tells you what is missing, label a
Jira ticket `agent-ready`, and get a plan PR.

See [`jira-agentic-workflow-summary.md`](jira-agentic-workflow-summary.md) for
what the pipeline currently does and why it is shaped this way.

## Current state

The pipeline *logic* is already portable. The staged flow, the label
lifecycle, the idempotency markers, the safe-output boundaries, and the
refuse-rather-than-guess actionability gate carry over to any team unchanged.

What is not portable is that instance configuration is fused into the prompt
prose. There is no seam between "the pipeline" and "this Jira site". Installing
it elsewhere today means a find-and-replace across five files with no way to
verify every occurrence was caught.

## Blocker inventory

### Instance identity hardcoded in prompt bodies

| Value | Locations |
| --- | --- |
| Cloud ID `15f7261f-…` | `jira-todo-ticket-planner.md:63`, `jira-plan-pr-handoff.md:67`, `jira-implementation-pr-handoff.md:71`, `jira-todo-console.md:33` |
| Site URL `https://dplewisdev.atlassian.net` | `jira-todo-ticket-planner.md:151` |
| Project key `FEAT` | `jira-todo-console.md:34` |

### Conventions hardcoded in both frontmatter and prose

- Status string `To Do`. Other boards use `Ready`, `Selected for Development`,
  or `Backlog`.
- Labels `agent-ready`, `agent-planned`, `plan-approved`, and
  `copilot-review-addressed` — roughly fifteen occurrences across
  `required-labels`, `if:` conditions, and prose checklists.
- The `plan/` branch prefix and `plans/*.md` path, appearing in
  `allowed-branches`, `paths` filters, `if:` guards, and every validation list.
- The exact title contract `<TICKET-KEY>: plan`, asserted by four workflows.

### Project-specific policy presented as universal

- The five-file actionability cap at `jira-todo-ticket-planner.md:120`. Some
  teams want two files, some want twenty.
- "Run the relevant existing validation" at
  `implement-approved-jira-plan.md:105` and
  `respond-to-copilot-review.md:118`. The agent has to guess the test command,
  and installing teams have no way to declare theirs.
- The agentic engine. Every workflow hardcodes `copilot-requests: true`, so a
  team standardised on Claude or Codex cannot adopt without editing all five.

### Defect surfaced by the review

The planner's JQL at `jira-todo-ticket-planner.md:75` has **no project
filter**:

```jql
status = "To Do" AND labels = "agent-ready" ORDER BY priority DESC, updated ASC
```

In a single-project sandbox this is invisible. In an organisation with many
Jira projects it will select another team's ticket. This must be fixed as part
of the extraction work, not deferred.

## Target architecture

### The configuration seam

Introduce `.github/aw/jira-pipeline.config.md`, imported by every pipeline
workflow, holding all instance values:

```markdown
## Pipeline configuration

- Jira cloud ID: `${{ vars.JIRA_CLOUD_ID }}`
- Jira site URL: `${{ vars.JIRA_SITE_URL }}`
- Jira project keys: `${{ vars.JIRA_PROJECT_KEYS }}`
- Ready status: `${{ vars.JIRA_READY_STATUS }}`
- Max files per plan: `${{ vars.PLAN_MAX_FILES }}`
- Validation command: `${{ vars.VALIDATION_COMMAND }}`
```

GitHub **repository variables** rather than secrets are the right home: these
values are not sensitive, and they can be set non-interactively by an install
script with `gh variable set`.

The planner already interpolates `${{ inputs.ticket }}` into its prompt body,
so expression interpolation into prompts works. Whether `vars` is permitted
under `strict: true` still needs verifying — gh-aw maintains an expression
allowlist, and everything else in this plan depends on the answer.

### What to parameterise versus fix

Parameterise the values a team genuinely differs on: cloud ID, site URL,
project keys, ready status, file cap, validation command, and engine.

Deliberately **do not** parameterise the branch prefix, plan path, or title
format. They appear in frontmatter positions such as `allowed-branches` and
`paths:` that cannot take runtime values, and the cost of working around that
exceeds the benefit. Document them instead as the convention the pipeline
owns: `plan/*` branches and `plans/**` files belong to this pipeline.

Per-project code conventions belong in `AGENTS.md`, which two workflows
already read but which does not exist in this repository. Shipping a template
gives teams a familiar place to declare test commands, file layout, and code
style, and removes the guesswork from the validation step.

### Package layout

```
.github/workflows/          the five pipeline workflows (the product)
.github/aw/
  jira-pipeline.config.md   the configuration seam
  instructions.md           repo overlay that SKILL.md already looks for
templates/AGENTS.md         starting point for installing teams
install/                    gh variable set, gh label create, secret checklist
docs/
```

### Distribution mechanism

`.github/skills/agentic-workflows/SKILL.md` already points at
`.github/aw/reuse.md` and `.github/aw/create-shared-agentic-workflow.md` in
`github/gh-aw`. Read both before building anything bespoke. gh-aw has
first-class support for consuming workflows from another repository, and that
is a much better adoption story than "clone and edit". Fall back to an
install script plus `gh aw compile` only if the shared-workflow path does not
cover this case.

### Repurposing the console workflow

`jira-todo-console.md` is currently development scaffolding, but it is most of
an install-time preflight already. Rename it `jira-pipeline-preflight` and
have it verify that:

- the `ATLASSIAN_MCP_BASIC` secret authenticates
- the configured cloud ID resolves
- the configured JQL returns results
- the required GitHub labels exist
- `GH_AW_CI_TRIGGER_TOKEN` has Contents write, if still required

This converts a silent, frustrating install failure into a diagnosable one,
and becomes the documented first step after configuration.

## Robustness gaps to close

These are single-operator assumptions rather than portability blockers, but
they surface immediately at team scale.

**No abandon path.** Nothing ever removes `agent-planned`. If a reviewer
closes a plan PR as unsatisfactory, the ticket is stranded — the planner will
never select it again. Add a reject flow: closing a plan PR restores
`agent-ready` or applies `agent-rejected`.

**No plan revision path.** The plan template writes `Plan v1`
(`jira-todo-ticket-planner.md:152`) but nothing ever produces a v2. A reviewer
who wants changes can only close and rerun.

**Implementation pushes may not trigger CI.** `implement-approved-jira-plan.md`
pushes without `github-token-for-extra-empty-commit`, whereas
`respond-to-copilot-review.md:31` uses it. Pushes made with `GITHUB_TOKEN` do
not fire `pull_request` workflows, so in a repository with real CI the
implementation commit may land unchecked and the human reviews an unverified
diff. Confirm the behaviour and make the two consistent.

**One secret is probably removable.** The plan handoff triggers on
`pull_request: [synchronize]`, which is the sole reason
`GH_AW_CI_TRIGGER_TOKEN` exists. The implementation handoff already
demonstrates the alternative: `workflow_run` keyed to a named workflow. Firing
the plan handoff off "Jira To Do Ticket Planner" completing would drop a
fine-grained PAT with Contents write from the install requirements — a
meaningful adoption barrier at most organisations.

**Blanket silent failure.** `report-failure-as-issue: false` is correct for
the empty-queue noop, but as a default across all five workflows it means an
installing team who is not watching the Actions tab never learns that anything
broke. Reconsider per workflow.

## Work sequence

1. **Extract configuration.** Create the config seam, move all instance values
   to repository variables, and fix the missing JQL project filter. Recompile
   all lockfiles. This phase also answers the `vars`-under-`strict` question
   that the rest depends on.
2. **Write the real README.** After the hardcoded cloud ID, the one-line
   README is the largest adoption blocker.
3. **Ship `templates/AGENTS.md`** and make validation-command declaration
   explicit rather than inferred.
4. **Convert the console workflow into preflight.**
5. **Add the abandon and reject path.**
6. **Choose and implement distribution** via gh-aw shared workflows.
7. **Add a LICENSE.** Nobody installs an unlicensed pipeline into their
   organisation.

Steps 1 through 5 are independently useful to this repository even if the
distribution work in step 6 is never done.

## Open questions

- Does `${{ vars.* }}` interpolate into prompt bodies under `strict: true`?
- Can the agentic engine be selected by configuration, or does it have to be a
  compile-time edit per installation?
- Should multi-project installs run one pipeline with a multi-key JQL, or one
  configured instance per Jira project?
- Is the Copilot review stage a required part of the package or an optional
  add-on? It depends on repository ruleset configuration that the installing
  team controls.

## Compiling after changes

Every workflow source has a generated `.lock.yml`. After changing frontmatter
or prompts:

```bash
gh aw compile <workflow-id> --strict --approve
```

Commit the source Markdown and its lockfile together.
