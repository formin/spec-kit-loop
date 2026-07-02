# spec-kit-loop quickstart

This repository is **Loop Engineering**, a [Spec Kit](https://github.com/github/spec-kit) community extension by `formin` (extension id `loop`, MIT license). It distributes 5 prompt-file commands — there is no executable code in this repository. The commands are markdown prompts that a user's coding agent (Claude Code, GitHub Copilot, Cursor, Gemini CLI, etc.) interprets; all state the commands manage is plain markdown written into the user's project. It operationalizes Addy Osmani's "Loop Engineering" essay for Spec Kit's build-and-verify phase.

## What this repository does

- Wraps Spec Kit's open-ended `/speckit.implement` step in a bounded, independently graded, human-signed-off loop; it never edits `spec.md`, `plan.md`, or `tasks.md`.
- Provides 5 slash commands: `/speckit.loop.define` (write the loop contract), `/speckit.loop.run` (maker), `/speckit.loop.check` (independent adversarial checker), `/speckit.loop.guard` (comprehension-debt sweep + human sign-off gate), `/speckit.loop.status` (read-only session resume).
- Enforces a maker/checker split: the maker never grades its own work; only the checker writes verdicts; only `guard` with a recorded human sign-off can set the loop to `done`.
- Externalizes all loop state to `specs/<feature>/loop/*.md` (`loop.md`, `iterations.md`, `verdicts.md`, `debt.md`, `memory.md`) so the loop survives session restarts and is diffable in PRs.
- Declares two optional hooks in `extension.yml`: `after_tasks` → `speckit.loop.define` and `after_implement` → `speckit.loop.check`.
- Requires Spec Kit `>=0.2.0`; no external tools, MCP servers, or network access.

## Start here

- New to the extension's design? Read [architecture/overview.md](architecture/overview.md) for the manifest schema, the prompt-file command mechanism, hooks, config precedence, and the loop state files.
- Looking up a specific command's behavior? See [commands/reference.md](commands/reference.md).
- Installing, configuring, or releasing? See [operations/installation-and-config.md](operations/installation-and-config.md).

## Documentation map

- [quickstart.md](quickstart.md) — this page: what the repository is and where everything lives.
- [architecture/overview.md](architecture/overview.md) — extension structure: manifest, prompt-file commands, hooks, config precedence, and the state files the loop creates in user projects.
- [commands/reference.md](commands/reference.md) — per-command reference: purpose, arguments, files read/written, and invariants for all 5 commands.
- [operations/installation-and-config.md](operations/installation-and-config.md) — installation options, configuration keys, the release process, and how these docs are refreshed by CI.

## Key source files

- [extension.yml](../extension.yml) — the extension manifest: identity, `requires`, the 5 command declarations, the config template declaration, 2 hooks, and all default settings.
- [commands/speckit.loop.define.md](../commands/speckit.loop.define.md) — prompt for writing the loop contract and creating the 5 state files.
- [commands/speckit.loop.run.md](../commands/speckit.loop.run.md) — prompt for the maker: one externalized increment per iteration, budget-gated.
- [commands/speckit.loop.check.md](../commands/speckit.loop.check.md) — prompt for the independent, adversarial checker with a durable verdict table.
- [commands/speckit.loop.guard.md](../commands/speckit.loop.guard.md) — prompt for the stay-the-engineer checkpoint and the only path to `done`.
- [commands/speckit.loop.status.md](../commands/speckit.loop.status.md) — prompt for the read-only, budget-aware state snapshot and session resume.
- [config-template.yml](../config-template.yml) — user-facing config template copied to `.specify/extensions/loop/loop-config.yml`.
- [docs/concepts.md](../docs/concepts.md) — the mapping from the Loop Engineering essay to this extension, and its honest limits as a prompt-only system.
- [README.md](../README.md) — full user-facing documentation: rationale, installation, tutorial, patterns, troubleshooting.
- [CHANGELOG.md](../CHANGELOG.md) — Keep a Changelog format; v1.0.0 (2026-06-14) is the initial release.
