# Making the Jira Pipeline Portable

## Goal

Turn this repository from a working single-instance experiment into an
installable package that another team can add to their own repository and
operate without editing prompt text.

The target install experience is: set a handful of repository variables and
two secrets, run a preflight workflow that tells you what is missing, label a
Jira ticket `agent-ready`, and get a plan PR.

See the [README](../README.md) for what the pipeline does, why it is shaped
that way, and the gh-aw behaviours that shaped it.

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

**Implemented.** `.github/workflows/shared/jira-pipeline-config.md` holds the
`env:` block and the shared policy prose, and each Jira-touching workflow pulls
it in with frontmatter:

```yaml
imports:
  - shared/jira-pipeline-config.md
```

GitHub **repository variables** rather than secrets are the right home for
these values: they are not sensitive, and an install script can set them
non-interactively with `gh variable set`.

Two compiler constraints shaped the final design, both established by probing
`gh aw compile --strict` (gh-aw v0.86.2) rather than assumed:

**`vars.*` is not allowed in prompt bodies.** Compilation fails with
`unauthorized expressions found`. The allowlist covers `github.event.*`,
`inputs.*`, `needs.*`, `steps.*`, and `env.*` — but not `vars.*`. The fix is to
bridge through frontmatter, which is ordinary Actions YAML and not subject to
the prompt allowlist:

```yaml
env:
  JIRA_CLOUD_ID: ${{ vars.JIRA_CLOUD_ID }}
  JIRA_READY_STATUS: ${{ vars.JIRA_READY_STATUS || 'To Do' }}
```

The prompt body then reads `${{ env.JIRA_CLOUD_ID }}`, which compiles. The
`|| 'default'` form works here, so a fresh install boots on sensible defaults.

**Imported markdown is not interpolated.** An `imports:` entry merges the
imported file's frontmatter (confirmed: the lockfile header attributes each
variable to `shared/jira-pipeline-config.md`) and prepends its prose to the
prompt via `{{#runtime-import}}`. But only the *main* workflow's own body has
its expressions hoisted into `GH_AW_ENV_*` placeholders. An expression written
in the imported file silently renders empty.

So the shared file carries the `env:` block and expression-free policy prose,
and each workflow repeats a short `## Configuration` section binding the values
it actually uses. This is verifiable in the lockfiles: every workflow now
hoists exactly the variables it references.

### What to parameterise versus fix

Parameterise the values a team genuinely differs on: cloud ID, site URL,
project keys, ready status, file cap, and engine.

The validation command was originally listed here as a repository variable.
It went to `AGENTS.md` instead: validation is rarely one command, often needs
ordering and caveats about toolchains or skipped suites, and belongs in the
file teams already use for exactly this. A single-line variable could not
carry that, and two homes for the same fact would be worse than one.

Deliberately **do not** parameterise the branch prefix, plan path, or title
format. They appear in frontmatter positions such as `allowed-branches` and
`paths:` that cannot take runtime values, and the cost of working around that
exceeds the benefit. Document them instead as the convention the pipeline
owns: `plan/*` branches and `plans/**` files belong to this pipeline.

The lifecycle labels stay literal for a stronger reason than convenience.
`required-labels`, the `labels:` trigger filter, and `add-labels.allowed` are
the pipeline's security boundary — they are what stops an unapproved plan from
being implemented. Keeping them as literals means the boundary is auditable by
reading the compiled lockfile, with no indirection through a variable an
installing team could later change without noticing what it gates.

Per-project code conventions belong in `AGENTS.md`, which two workflows
already read but which does not exist in this repository. Shipping a template
gives teams a familiar place to declare test commands, file layout, and code
style, and removes the guesswork from the validation step.

### Package layout

```
.github/workflows/
  shared/
    jira-pipeline-config.md   the configuration seam (implemented)
  *.md                        the five pipeline workflows (the product)
.github/aw/
  instructions.md             repo overlay that SKILL.md already looks for
templates/AGENTS.md           starting point for installing teams
install/                      gh variable set, gh label create, secret checklist
docs/
```

The shared file lives under `.github/workflows/shared/` rather than
`.github/aw/` because `imports:` paths resolve relative to the workflows
directory.

### Distribution mechanism

`.github/skills/agentic-workflows/SKILL.md` already points at
`.github/aw/reuse.md` and `.github/aw/create-shared-agentic-workflow.md` in
`github/gh-aw`. Read both before building anything bespoke.

The CLI confirms first-class support for cross-repository distribution, which
is a much better adoption story than "clone and edit":

| Command | Purpose |
| --- | --- |
| `gh aw add` | Add workflows from another repository, a local file, or a URL |
| `gh aw add-wizard` | Interactive guided install |
| `gh aw deploy` | Deploy workflows into a target repository via pull request |
| `gh aw update` | Update installed workflows from their source repository |
| `gh aw secrets` | Manage the Actions secrets a workflow requires |

`gh aw update` matters most: it means installing teams get fixes without
re-copying files. Design the package so that everything a team edits lives in
repository variables and `AGENTS.md`, never in the workflow bodies themselves,
or updates will clobber local changes.

### Repurposing the console workflow

**Implemented.** `jira-todo-console.md` was development scaffolding; it is now
`jira-pipeline-preflight.md`, which verifies that:

- the three required repository variables are set, and what the optional ones
  resolve to
- both secrets are present, and that `ATLASSIAN_MCP_BASIC` actually
  authenticates against the configured site
- the `plan-approved` and `copilot-review-addressed` labels exist
- GitHub Actions is permitted to create pull requests
- a root `AGENTS.md` exists, so changes are validated rather than assumed
- the scoped JQL returns results

This converts a silent, frustrating install failure into a diagnosable one,
and becomes the documented first step after configuration.

Two design points worth keeping if this workflow is rewritten:

**The checks run in a `steps:` block, not in the prompt.** Everything except
live Jira connectivity is a deterministic shell check that writes JSON and a
job summary before the agent starts. The agent interprets and reports; it does
not decide whether a label exists. Configuration diagnosis is precisely the
thing that must not depend on model judgement.

**A healthy repository files no issue.** gh-aw injects an auto-create-issue
fallback whenever a workflow declares safe outputs, and no frontmatter toggle
suppresses it — verified by probing `create-issue: false`, `noop: true`, and
several explicit output declarations, none of which removed
`safe_outputs_auto_create_issue` from the compiled prompt. The original console
workflow therefore filed an issue on every successful run. The preflight
instead instructs `noop` on a ready result, so only a real problem produces an
issue.

## Robustness gaps to close

These are single-operator assumptions rather than portability blockers, but
they surface immediately at team scale.

~~**No abandon path.**~~ **Closed.** `jira-plan-pr-closed.md` now fires on
`pull_request: closed` for `plan/*` branches. A merged PR removes
`agent-planned` and records delivery; a PR closed unmerged replaces it with
`agent-rejected` and explains how to re-enter the ticket.

The reject path deliberately does **not** restore `agent-ready`, which was the
original suggestion. Doing so would loop: the planner would regenerate the same
plan from the same unchanged ticket, and a person would have to close it again.
`agent-rejected` is terminal until a human swaps the label back.

**No plan revision path.** The plan template writes `Plan v1`
(`jira-todo-ticket-planner.md:152`) but nothing ever produces a v2. A reviewer
who wants changes can only close and rerun.

**Implementation pushes may not trigger CI.** `implement-approved-jira-plan.md`
pushes without `github-token-for-extra-empty-commit`, whereas
`respond-to-pr-comment.md` uses it. Pushes made with `GITHUB_TOKEN` do
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
the empty-queue noop, but as a default across every workflow it means an
installing team who is not watching the Actions tab never learns that anything
broke. Reconsider per workflow.

## Work sequence

1. ~~**Extract configuration.**~~ **Done.** Config seam created, all instance
   values moved to repository variables, JQL project filter fixed, all seven
   workflows recompiled. Both compiler constraints are documented above.
2. ~~**Write the real README.**~~ **Done.** Install steps, ticket contract,
   operating procedure, configuration reference, safety model, and
   troubleshooting. Documented CLI invocations are verified against
   `gh aw` v0.86.2 rather than assumed.
3. ~~**Ship `templates/AGENTS.md`**~~ **Done.** Template added, plus this
   repository's own `AGENTS.md`, which two workflows had always looked for but
   which never existed. Validation is now declared in `AGENTS.md` rather than
   inferred, and — the substantive fix — an agent that runs no validation must
   say so verbatim instead of filling the `### Validation` section with prose
   that reads like verification.
4. ~~**Convert the console workflow into preflight.**~~ **Done.** Deterministic
   checks in a `steps:` block, live Jira check by the agent, ready/degraded/
   blocking classification, and no issue filed when the repository is healthy.
5. ~~**Add the abandon and reject path.**~~ **Done.** Both outcomes handled,
   with `agent-rejected` as a terminal state rather than an automatic
   re-plan.
6. **Choose and implement distribution** via gh-aw shared workflows.
7. **Add a LICENSE.** Nobody installs an unlicensed pipeline into their
   organisation.

Steps 1 through 5 are independently useful to this repository even if the
distribution work in step 6 is never done.

## Open questions

- ~~Does `${{ vars.* }}` interpolate into prompt bodies under `strict: true`?~~
  **Answered: no.** Bridge through frontmatter `env:` and reference `env.*`.
  Imported files are not interpolated either; see the configuration seam
  section.
- Can the agentic engine be selected by configuration, or does it have to be a
  compile-time edit per installation?
- Should multi-project installs run one pipeline with a multi-key JQL, or one
  configured instance per Jira project?
- Is the Copilot review stage a required part of the package or an optional
  add-on? It depends on repository ruleset configuration that the installing
  team controls.

## Repository variables

Set these with `gh variable set <NAME> --body "<value>"`.

| Variable | Required | Default | Example |
| --- | --- | --- | --- |
| `JIRA_CLOUD_ID` | yes | — | `15f7261f-…` |
| `JIRA_SITE_URL` | yes | — | `https://acme.atlassian.net` |
| `JIRA_PROJECT_KEYS` | yes | — | `FEAT` or `FEAT,PLAT` |
| `JIRA_READY_STATUS` | no | `To Do` | `Selected for Development` |
| `PLAN_MAX_FILES` | no | `5` | `8` |

The three required variables have no default on purpose. The configuration
gate makes each workflow `noop` and name the unset variable rather than fall
back to a guess — which is what previously allowed an unscoped, cross-project
Jira search.

## Compiling after changes

Every workflow source has a generated `.lock.yml`. After changing frontmatter
or prompts:

```bash
gh aw compile <workflow-id> --strict --approve
```

Commit the source Markdown and its lockfile together.
