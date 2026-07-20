# Docs Skills — Project Configuration

This file is read by the `design-doc`, `plan-feature`, `execute-plan`, and `audit-plan` skills (and
`update-arch-docs`, if present) to adapt their behavior to this project. Skills look for it at
`.claude/docs-skills.md` in the project root. Edit values here to re-tune the workflow — you should
never need to edit the skills themselves.

## Paths

| Key | Value |
|-----|-------|
| `plans-dir` | `{{PLANS_DIR}}` |
| `backlog-dir` | `{{BACKLOG_DIR}}` |
| `arch-docs-dir` | `{{ARCH_DOCS_DIR}}` |
| `audited-plans-dir` | `{{AUDITED_PLANS_DIR}}` |
| `design-docs-dir` | `{{DESIGN_DOCS_DIR}}` |

<!--
  design-docs-dir : the upstream design/spec corpus `design-doc` maintains, and that `plan-feature`
                    plans *from* and `audit-plan` checks the code *against*. Set it (e.g. docs/design/)
                    to turn the design→plan→code chain on; set `none` to turn it off (plan-feature and
                    audit-plan then behave as if the design corpus didn't exist).
-->

## Workflow

| Key | Value |
|-----|-------|
| `main-branch` | `{{MAIN_BRANCH}}` |
| `branch-prefix` | `{{BRANCH_PREFIX}}` |
| `pr-tool` | `{{PR_TOOL}}` |
| `worker-model` | `{{WORKER_MODEL}}` |
| `uses-worktrees` | `{{USES_WORKTREES}}` |
| `has-update-arch-docs-skill` | `{{HAS_UPDATE_ARCH_DOCS}}` |
| `shell` | `{{SHELL}}` |

<!--
  pr-tool         : gh | glab | none
  worker-model    : the model phase agents run on (sonnet is a good default; orchestration stays
                    on the session model). Any valid model id, or `inherit` to use the session model.
  uses-worktrees  : true if agent work happens in .claude/worktrees/<name> dirs (affects audit-plan
                    cleanup); false otherwise.
  shell           : bash | pwsh — which syntax the skills use for filesystem commands.
-->

## Testing

| Key | Value |
|-----|-------|
| `test-command` | `{{TEST_COMMAND}}` |
| `test-filter-template` | `{{TEST_FILTER_TEMPLATE}}` |
| `tests-dir` | `{{TESTS_DIR}}` |
| `tests-are-slow` | `{{TESTS_ARE_SLOW}}` |

<!--
  test-filter-template : command to run a single test/class by name; use {NAME} as the placeholder
                         execute-plan substitutes (e.g. dotnet test --filter "FullyQualifiedName~{NAME}",
                         npx jest -t "{NAME}", pytest -k "{NAME}", cargo test {NAME}). Leave blank if
                         the runner can't filter by name.
  tests-are-slow       : true  -> execute-plan runs the full suite twice (discover, confirm) and uses
                                  the filter template to iterate on failures in between.
                         false -> the suite is cheap; just run it fully each time.
-->

## Cross-cutting invariants

These rules must be maintained throughout any plan's implementation phases. `execute-plan` injects
them verbatim into every phase agent's prompt, so keep them concrete and imperative. Add or remove
freely — this section is the heart of what makes the workflow fit *this* codebase.

{{INVARIANTS}}
