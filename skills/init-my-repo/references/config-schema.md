# `.claude/docs-skills.md` — config schema reference

The authoritative list of keys the workflow skills read, their defaults, and how to detect a value
during `init-my-repo`. Written into each repo by `init-my-repo` from
`assets/docs-skills.template.md`. All the skills degrade gracefully when a key is missing — they
fall back to the default and, where the default can't be right (e.g. `test-command`), ask the user.

## Paths

| Key | Meaning | Default | How to detect |
|-----|---------|---------|---------------|
| `plans-dir` | where plans are written/read | `docs/plans/` | glob existing `**/plans/`, match repo casing |
| `backlog-dir` | deferred/out-of-scope items (plan-feature Step 3.5) | `docs/backlog/` | glob `**/backlog/` |
| `arch-docs-dir` | architecture docs (ground-truth reading + Step 5 updates) | `docs/architecture/` | glob `**/architecture/` or `**/adr/` |
| `audited-plans-dir` | archive for fully-implemented plans (audit-plan Step 5) | `<plans-dir>/../old_plans/audited/` | glob `**/old_plans/`, else derive from `plans-dir` |
| `design-docs-dir` | upstream design corpus that `design-doc` maintains and `plan-feature`/`audit-plan` trace against | `none` (workflow off) | glob `**/design/`, `**/spec/`, `**/rfcs/`; if `design-doc` skill is installed, default `docs/design/` |

### The design-corpus workflow (`design-docs-dir`)

`design-docs-dir` wires up the optional **design → plan → code** chain that the `design-doc` skill
anchors. It behaves differently from the other paths because it is a *feature switch*, not just a
location:

- **Set** (e.g. `docs/design/`): `plan-feature` reads the relevant design doc as the *source of
  intent* and records a **Design traceability** section in the plan; `audit-plan` then checks the
  code against that design, not just against the plan. This is the mode `init-my-repo` writes
  whenever it installs `design-doc`.
- **`none` or absent**: the two plan skills skip every design step and behave exactly as they did
  before this key existed. This is what keeps the chain non-breaking for repos that don't keep a
  design corpus.

Note the asymmetry in defaults: `design-doc` itself defaults to `docs/design/` when the key is
missing (it *is* the corpus's keeper, so it needs somewhere to write), whereas `plan-feature` and
`audit-plan` default to **off** (they must not go hunting for a corpus a repo never opted into).

## Workflow

| Key | Meaning | Default | How to detect |
|-----|---------|---------|---------------|
| `main-branch` | branch never committed to directly | `main` | `git symbolic-ref refs/remotes/origin/HEAD`, else `main`/`master` presence |
| `branch-prefix` | prefix for a feature branch execute-plan may create | `plan/` | ask, or infer from existing branch names |
| `pr-tool` | `gh` / `glab` / `none` | `gh` if on PATH else `none` | `gh --version`, then `glab --version` |
| `worker-model` | model for spawned phase agents; `inherit` = session model | `sonnet` | ask; sonnet is a good default |
| `milestone-model` | model for the per-milestone orchestrator agents `execute-umbrella` spawns | `opus` | ask; it runs a whole `execute-plan` each, so give it a capable one |
| `uses-worktrees` | agent work happens in `worktrees-dir` (audit-plan cleanup) | `true` | glob `.claude/worktrees/` |
| `worktrees-dir` | where `execute-umbrella` creates one worktree per milestone | `.claude/worktrees/` | glob for an existing convention; must be gitignored |
| `has-update-arch-docs-skill` | delegate docs to `update-arch-docs` vs. update inline | `false` | glob `.claude/skills/update-arch-docs/` and `~/.claude/skills/update-arch-docs/` |
| `shell` | `bash` / `pwsh` — syntax for filesystem commands | OS-based (win → `pwsh`) | host OS |

## Testing

| Key | Meaning | Default | How to detect |
|-----|---------|---------|---------------|
| `test-command` | full-suite test command | ask | manifest file (table below) |
| `test-filter-template` | run a subset by name; `{NAME}` is the substitution point | none | per runner (table below) |
| `tests-dir` | where new test files go | infer | glob `**/[Tt]ests/`, `**/__tests__/`, `**/test/` |
| `tests-are-slow` | run-twice-and-narrow (true) vs. run-fully-each-time (false) | infer | true for compiled/integration suites; false for fast unit suites |

### Per-runner detection

| Manifest | `test-command` | `test-filter-template` |
|----------|----------------|------------------------|
| `*.csproj` / `*.sln` | `dotnet test` | `dotnet test --filter "FullyQualifiedName~{NAME}"` |
| `package.json` (jest) | `npm test` | `npx jest -t "{NAME}"` |
| `package.json` (vitest) | `npm test` | `npx vitest run -t "{NAME}"` |
| `pyproject.toml` / `pytest.ini` | `pytest` | `pytest -k "{NAME}"` |
| `Cargo.toml` | `cargo test` | `cargo test {NAME}` |
| `go.mod` | `go test ./...` | `go test ./... -run "{NAME}"` |
| `pom.xml` | `mvn test` | `mvn test -Dtest="{NAME}"` |
| `build.gradle` | `./gradlew test` | `./gradlew test --tests "{NAME}"` |

Read `package.json`'s `scripts.test` when present — the real command may differ from `npm test`
(e.g. `vitest run`, `jest --config …`). Prefer the actual script.

## Cross-cutting invariants

Free-form prose section (not a table), injected verbatim into every `execute-plan` phase agent.
Harvest candidates from the repo's `CLAUDE.md` — look under headings like "Non-negotiable rules",
"Conventions", "Architecture", "Rules". Good invariants are concrete and imperative:

- threading / concurrency constraints ("no UI-thread calls in the simulation layer")
- where configuration lives ("gameplay tuning is JSON in `Config/`, not code")
- a style/architecture doc to follow ("follow SOLID as described in `Docs/SOLID_Principles.md`")
- hygiene rules ("delete dead code as you go; don't leave the old path beside the new one")
- testing conventions ("new subsystems get tests when built; hand-written stubs, no mock framework")

If the user has none, it's fine to write a single line: "Follow the conventions already established
in the surrounding code." Empty is better than vague filler.

## Backward compatibility

Repos that already have a `.claude/docs-skills.md` (e.g. one written for an earlier version of these
skills) may carry keys not listed here — `ui-docs-dir`, `arch-template`, etc. When merging, **never
drop an unrecognized key**; another skill may depend on it. Update the values the user changed, add
the new keys, and leave the rest.
