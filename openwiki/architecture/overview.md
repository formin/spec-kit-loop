# Architecture overview

Loop Engineering is a **prompt-only Spec Kit extension**: it ships a manifest, 5 markdown command prompts, a config template, and docs. Nothing in this repository executes. Behavior comes from the user's coding agent reading a command prompt and following its steps, and from Spec Kit's `specify` CLI installing the files. The strongest guarantee this design can offer is stated in [docs/concepts.md](../../docs/concepts.md): rules are prompt-enforced, so drift is visible in diffs but not impossible — the extension makes skipping a guardrail an explicit, recorded choice rather than a silent default.

## Extension manifest (`extension.yml`)

[extension.yml](../../extension.yml) uses `schema_version: "1.0"` and has five top-level sections:

- `extension` — identity: `id: loop`, `name: "Loop Engineering"`, `version: "1.0.0"`, `category: "process"`, `effect: "read-write"`, author/repository/license/homepage.
- `requires` — `speckit_version: ">=0.2.0"`.
- `provides.commands` — the 5 commands, each mapping a `name` (e.g. `speckit.loop.define`) to a `file` under `commands/` plus a description. `provides.config` declares `loop-config.yml` generated from `config-template.yml`, marked `required: false`.
- `hooks` — two optional hooks (see below).
- `defaults` — the built-in settings for `loop`, `checker`, `guard`, `rendering`, and `state` (mirrored by `config-template.yml`).

## Command mechanism: prompts, not code

Each file under [commands/](../../commands/) is a markdown prompt with YAML frontmatter (a `description` field only). Spec Kit installs them as agent slash commands; when the user runs `/speckit.loop.run`, the agent receives the file body as instructions. Shared conventions across all 5 prompts:

- A `## User Input` section containing a literal `$ARGUMENTS` placeholder that Spec Kit substitutes with whatever the user typed after the command.
- Structured `## Steps` the agent follows, and a `## Guardrails` section of hard rules (e.g. "the maker never writes to `verdicts.md`").
- State resolution by convention: every command resolves `LOOP_DIR` the same way as `define` — feature directory from the current git branch matching `specs/<NNN-name>/`, else the most recently modified `specs/` directory, else the fallback `.specify/loops/global/`; then `LOOP_DIR = FEATURE_DIR/loop`.
- Role separation enforced by which command may write which file: `run` writes `iterations.md`/`memory.md`, `check` writes `verdicts.md`, `guard` writes the sign-off log in `debt.md`, `status` writes nothing.

## Hooks

Both hooks are declared in `extension.yml` and are `optional: true` — Spec Kit prompts the user before running them:

- `after_tasks` → `speckit.loop.define`: offer to turn a fresh `tasks.md` into a loop contract.
- `after_implement` → `speckit.loop.check`: offer an independent checker pass on the implementation.

## Configuration precedence

Lowest to highest (documented in `README.md` and the `define` prompt):

1. Extension defaults in `extension.yml` (`defaults:` block).
2. User config file `.specify/extensions/loop/loop-config.yml` (copied from [config-template.yml](../../config-template.yml); every key optional).
3. `SPECKIT_LOOP_*` environment variables (e.g. `SPECKIT_LOOP_LOOP_MAX_ITERATIONS=12`).
4. Per-invocation `key=value` arguments to `/speckit.loop.define` (e.g. `max_iterations=12 isolation=none`).

## Loop state files created in user projects

`/speckit.loop.define` creates 5 markdown files in `specs/<feature>/loop/` (or `.specify/loops/global/` when no feature directory exists). They are ordinary markdown in the user's repo — diffable and reviewable in PRs:

- `loop.md` — the contract and live status: numbered purpose(s), a done-criteria table (`ID | Criterion | How the checker verifies it | Status`), budget (max iterations, iterations run, isolation), roles, allowed tools/connectors, automation trigger, guardrail settings, and `Phase` (`defined → maker-ready → checked → done`). Criterion statuses: `pending → maker-ready → checker-pass | checker-fail → human-signed`. Only `guard` may set `Phase: done`.
- `iterations.md` — the maker's append-only memory: one record per `run` iteration (targeted criteria, worktree path, change with file pointers, maker self-assessment, open risks, `Handoff: ready-for-check`). The maker never records a verdict here.
- `verdicts.md` — the checker's verdict table (`ID | Iteration | Criterion | Method (primary source) | Verdict | Confidence | Date`), written only by `check`.
- `debt.md` — the comprehension-debt ledger (open-debt table with severity) plus the sign-off log. Debt is acknowledged, never deleted; sign-offs are recorded human actions.
- `memory.md` — short durable `M-xxx` entries (decisions, conventions, dead ends) carried across runs so the loop does not re-derive them.

When `loop.isolation: worktree` (the default), `run` makes each iteration in a dedicated git worktree (e.g. `git worktree add ../loop-iter-<n> <branch>`) and records the path in the iteration record.
