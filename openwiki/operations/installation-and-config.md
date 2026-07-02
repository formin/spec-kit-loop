# Installation, configuration, and releases

## Installation

Loop Engineering is listed in the [Spec Kit community extension catalog](https://github.com/github/spec-kit/blob/main/extensions/catalog.community.json). Requires Spec Kit `>=0.2.0`; works with any agent Spec Kit supports. Three install paths (see [README.md](../../README.md) for full detail):

- **By name from the community catalog.** The community catalog is discovery-only by default, so first set `install_allowed: true` for it in `.specify/extension-catalogs.yml` (per project) or `~/.specify/extension-catalogs.yml` (per user), then:

  ```bash
  specify extension add loop
  ```

- **From a pinned release URL** (zero catalog config):

  ```bash
  specify extension add loop --from https://github.com/formin/spec-kit-loop/archive/refs/tags/v1.0.0.zip
  ```

  URL installs show an *Untrusted Source* warning and prompt `Continue with installation? [y/N]`; answer `y` (non-interactive: `echo y | specify extension add …`). Catalog installs skip the prompt.

- **For development** from a local clone:

  ```bash
  git clone https://github.com/formin/spec-kit-loop
  specify extension add --dev ./spec-kit-loop
  ```

Verify with `specify extension list` — expect `Loop Engineering (v1.0.0)`, `Commands: 5 | Hooks: 2`.

## Configuration (`config-template.yml`)

Copy [config-template.yml](../../config-template.yml) to `.specify/extensions/loop/loop-config.yml`. Every key is optional; missing keys fall back to the `defaults:` block in [extension.yml](../../extension.yml). Precedence (lowest → highest): extension defaults → config file → `SPECKIT_LOOP_*` environment variables (e.g. `SPECKIT_LOOP_LOOP_MAX_ITERATIONS=12`) → `key=value` arguments to `/speckit.loop.define`.

- `loop.max_iterations` (default `8`) — hard ceiling on maker iterations before forced human review.
- `loop.done_requires_checker_pass` (default `true`) — the loop cannot reach `done` without a passing checker verdict.
- `loop.isolation` (default `worktree`; `worktree | none`) — whether each maker iteration runs in its own git worktree.
- `checker.independent` (default `true`) — refuse to grade if the checker shares the maker's context.
- `checker.adversarial` (default `true`) — the checker tries to break the work and defaults to `fail` when uncertain.
- `checker.votes` (default `1`) — independent checker passes that must agree before a criterion passes.
- `guard.require_human_signoff` (default `true`) — a human must explicitly sign off before `done`.
- `guard.track_comprehension_debt` (default `true`) — record every change a human has not yet understood.
- `guard.block_done_on_open_debt` (default `true`) — open blocking debt prevents declaring done.
- `rendering.iterations_slice` (default `6`) / `rendering.debt_slice` (default `8`) — how many recent iterations / open debt items `/speckit.loop.status` shows.
- `rendering.context_tokens` (default `4000`) — soft cap for loop state rendered into context per step.
- `state.directory` (default `loop`) — subdirectory of the active feature dir (`specs/NNN-name/loop/`).
- `state.fallback_directory` (default `.specify/loops/global`) — used when no feature directory exists.

## Releasing a new version

The `--from` install path and the catalog both resolve to GitHub release archives, so a release is a tag plus a GitHub release:

1. Bump `extension.version` in [extension.yml](../../extension.yml) and add a section to [CHANGELOG.md](../../CHANGELOG.md) (Keep a Changelog / SemVer).
2. Tag the commit `vX.Y.Z` and push the tag.
3. Create a GitHub release for the tag — the auto-generated source archive (`.../archive/refs/tags/vX.Y.Z.zip`) is what `specify extension add loop --from <url>` installs.
4. Update install snippets in `README.md` that pin the old version URL.

## OpenWiki refresh workflow

The `openwiki/` docs are refreshed **daily at 08:00 KST by a local scheduled
task on the maintainer's machine**, running `openwiki --update --print`
against a local LM Studio server (`OPENWIKI_PROVIDER=openai`,
`OPENAI_BASE_URL=http://127.0.0.1:1234/v1`) and pushing any `openwiki/`
changes directly to `main` (`docs: update OpenWiki`). If the LM Studio server
is not running, the task logs the fact and skips the run.

[.github/workflows/openwiki-update.yml](../../.github/workflows/openwiki-update.yml)
remains as a **manual cloud fallback** (`workflow_dispatch` only — GitHub
runners cannot reach the local endpoint): it runs OpenWiki with
`OPENWIKI_PROVIDER: anthropic` / `OPENWIKI_MODEL_ID: anthropic/claude-sonnet-5`
and opens a pull request (branch `openwiki/update`, via
`peter-evans/create-pull-request`). It requires an `ANTHROPIC_API_KEY`
repository secret and has `contents: write` and `pull-requests: write`
permissions.
