# AGENTS.md

Copy this file to your repository root and fill it in. The Jira pipeline reads
it at planning, implementation, and review-response time.

Delete the instructional comments as you go — everything left in this file is
read as fact about your project, so an unedited placeholder is worse than an
absent section.

---

## Validation

**This section is the one that matters most.** It tells the agent how to prove
a change works. Without it, the agent falls back to guessing from repository
conventions, and if it finds nothing it must report that no validation ran —
which means a human reviews an unverified diff.

List the exact commands, in the order they should run:

```bash
# Example — replace entirely
npm ci
npm run lint
npm test
```

Notes for whoever fills this in:

- Give commands that work from a clean checkout. The agent runs in a fresh
  container with no warm cache and no prior install.
- If a command needs a toolchain that is not present by default, say so here.
  The agent cannot install system-level dependencies.
- If some suites are too slow or need credentials, name the subset that is
  safe and fast to run, and say explicitly what is being skipped.
- If this project genuinely has no automated validation, write that sentence
  here. An explicit "there is no test suite; verify by inspection" is useful.
  Silence is not.

## Project layout

Where things live, so the agent inspects the right areas instead of reading
the whole tree.

```
src/           # application code
tests/         # test suites
docs/          # documentation
```

## Conventions

Anything a new contributor would get wrong on their first PR:

- Language, framework, and version constraints
- Naming, file organisation, and module boundaries
- Formatting and linting rules that are enforced
- Patterns to follow, and patterns that are deprecated
- Commit message format, if enforced

## Testing conventions

- Where tests live relative to the code they cover
- Naming pattern for test files
- The framework in use, and any required fixtures or factories
- Whether new code is expected to ship with tests (say so plainly — the agent
  will follow this)

## Out of bounds

Areas the agent must not modify, beyond the pipeline's built-in exclusions for
workflow files, credentials, and environment files:

- Generated files and their source of truth
- Vendored or third-party directories
- Anything requiring a migration, a coordinated deploy, or a review from a
  specific team

## Domain notes

Terminology, invariants, and non-obvious constraints that are not visible in
the code — the things a reviewer would flag that a newcomer could not have
known.
