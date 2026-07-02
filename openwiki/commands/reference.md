# Command reference

All 5 commands are markdown prompts under [commands/](../../commands/), executed by the user's coding agent. Each resolves `LOOP_DIR` the same way: feature directory matched from the current git branch under `specs/`, else the most recently modified `specs/` directory, else `.specify/loops/global/`.

## /speckit.loop.define

Source: [commands/speckit.loop.define.md](../../commands/speckit.loop.define.md)

- **Purpose**: write the loop as a contract — purpose (the recursive goal), checkable done-criteria, budget, maker/checker roles, allowed tools, isolation, automation trigger, guardrails — and create the state files.
- **Arguments**: free text is the loop's purpose; inline `key=value` tokens (e.g. `max_iterations=12 isolation=none`) are budget/policy overrides with highest precedence. Both may appear together.
- **Reads**: `.specify/extensions/loop/loop-config.yml` (if present), `SPECKIT_LOOP_*` env vars, and the active `spec.md`/`tasks.md` to help elicit the contract.
- **Writes**: creates `LOOP_DIR` with `loop.md`, `iterations.md`, `verdicts.md`, `debt.md`, `memory.md` — each exactly once.
- **Invariants**: refuses to create a loop without checkable done-criteria ("it works" is not a criterion; "`npm test` exits 0" is). Idempotent — if `loop.md` exists it never overwrites; it reports status instead, and a new purpose is appended as an additional numbered goal. Does not start producing work.

## /speckit.loop.run

Source: [commands/speckit.loop.run.md](../../commands/speckit.loop.run.md)

- **Purpose**: the **maker** — produce one smallest coherent increment toward a targeted criterion, externalize it, and stop with a handoff to the checker.
- **Arguments** (optional): specific criterion IDs (e.g. `D2 D3`), or `n=<k>` to run up to `k` iterations back-to-back before a single checker handoff. Default: highest-priority `pending`/`checker-fail` criteria, preferring `checker-fail`.
- **Reads**: `loop.md` (contract, counter, `max_iterations`), the last few `iterations.md` records, `memory.md` — files as source of truth, never conversation history.
- **Writes**: appends an iteration record to `iterations.md`; updates `loop.md` (increments `Iterations run`, moves targeted criteria to `maker-ready`, sets `Phase: maker-ready`); adds an `M-xxx` line to `memory.md` when a durable decision was made.
- **Invariants**: budget gate — if `Iterations run >= max_iterations`, stops and demands `check`/`guard` instead of more making. Never writes `verdicts.md`, never sets `checker-pass` or `done`. Stays inside the contract's allowed-tools list — stops rather than reach for an unlisted tool. Uses a per-iteration git worktree when `loop.isolation: worktree`. Self-assessment is always labelled as the maker's own view.

## /speckit.loop.check

Source: [commands/speckit.loop.check.md](../../commands/speckit.loop.check.md)

- **Purpose**: the **independent checker** — adversarially grade the latest iteration against the done-criteria at primary sources (run the test, hit the endpoint, open the file) and record durable verdicts.
- **Arguments** (optional): specific criterion IDs or an iteration number. Default: every criterion currently `maker-ready`, for the latest iteration.
- **Reads**: `loop.md` and the latest `iterations.md` record (treated as a claim to be tested, not evidence). Requires a defined loop with at least one iteration.
- **Writes**: appends verdict rows to `verdicts.md`; updates criterion statuses in `loop.md` (`pass` → `checker-pass`, `fail` → `checker-fail`, `uncertain` → stays `maker-ready`) and sets `Phase: checked`; opens a `debt.md` item for each `uncertain` criterion.
- **Invariants**: independence gate — with `checker.independent: true`, if this session is the same one that produced the iteration, it says so and asks to be re-run in a fresh session or sub-agent rather than rubber-stamp. Tries to make each criterion fail; defaults to `fail`/`uncertain` when unsure. With `checker.votes > 1`, that many independent passes must agree or the result is `uncertain`. Never edits the implementation, never writes `iterations.md`. A clean sweep routes to `guard`, never declares `done`.

## /speckit.loop.guard

Source: [commands/speckit.loop.guard.md](../../commands/speckit.loop.guard.md)

- **Purpose**: the stay-the-engineer checkpoint — comprehension-debt sweep, unattended-verification flags, anti-cognitive-surrender questions, and the human sign-off gate. The **only** command that can move the loop to `done`.
- **Arguments** (optional): `signoff` records a human sign-off; a criterion ID scopes the sign-off. No argument renders the guard report only.
- **Reads**: `loop.md`, `verdicts.md`, `debt.md`, and `iterations.md` records since the last recorded sign-off.
- **Writes**: new debt rows in `debt.md` for unreviewed changes; on `signoff`, a row in the `## Sign-off log` and acknowledgment of relevant debt rows; sets `loop.md` `Phase: done` only when the full done-gate holds.
- **Invariants**: done requires all three — every criterion `checker-pass` (when `done_requires_checker_pass`), no open blocking (high-severity) debt (when `block_done_on_open_debt`), and a recorded human sign-off (when `require_human_signoff`). Never fabricates a sign-off — it logs a human action. Flags `checker-pass` at medium/low confidence or from a possibly non-independent checker as unattended verification. Debt is acknowledged, never silently deleted. Does not write code or grade criteria.

## /speckit.loop.status

Source: [commands/speckit.loop.status.md](../../commands/speckit.loop.status.md)

- **Purpose**: render a compact, budget-aware snapshot of loop state with exactly one recommended next action; the session-resume entry point after a restart or compaction.
- **Arguments** (optional): `full` renders 3× the configured slice sizes; a criterion ID or topic filters the tables.
- **Reads**: `loop.md`; the last `rendering.iterations_slice` records of `iterations.md`; latest verdict per criterion from `verdicts.md`; up to `rendering.debt_slice` open debt rows and the most recent sign-off from `debt.md`. Does not read `memory.md` bodies or spec/plan/code files; keeps output within the `rendering.context_tokens` cap.
- **Writes**: nothing — strictly read-only, not even timestamps. If no loop exists it points to `define` without creating anything.
- **Invariants**: slices, never full files — counts stand in for what is not shown. The single recommendation must follow from the rendered state (`checker-fail`/`pending` → `run`; `maker-ready` ungraded → `check` in a separate session; all `checker-pass` with gate outstanding → `guard`; ceiling reached → `guard`, not more `run`; all conditions met → done). Reports missing/malformed state files instead of guessing.
