# AGENTS.md

## Repo Shape
- `main` is documentary/shared infrastructure; runnable workflows live on branches such as `two-pack`, `four-pack`, and `six-pack`.
- Core launcher code is Babashka in `swarmforge/scripts/swarmforge.bb`; `swarmforge/scripts/swarmforge.sh` is only the zsh wrapper that execs it.
- Handoff behavior is split between `swarm_handoff.bb`, `handoffd.bb`, `handoff_lib.bb`, and the `ready_for_next*` / `done_with_current*` helpers.
- `swarmforge/constitution/articles/*.prompt` are shipped shared articles copied into runnable branches/worktrees; do not casually rewrite them as branch-local workflow instructions.

## Commands
- Run the full test suite with `bb test` from the repo root.
- Focused launcher parsing checks use `swarmforge/scripts/swarmforge.bb --test-parse <fixture-root>`; this prepares `.swarmforge/` state but does not launch agents.
- Other focused test hooks are in `swarmforge.bb`: `--test-terminal-bridge`, `--test-launch-command`, `--test-agent-start-delay`, and `--test-tmux-base-indexes`.

## Runtime And Generated State
- Swarm runtime state is local and ignored: `.swarmforge/` and `.worktrees/` must not be staged or committed.
- Startup writes `.swarmforge/roles.tsv`, `sessions.tsv`, `tmux-socket`, prompt files, daemon logs, handoff queues, and copied helper scripts into role worktrees.
- The launcher ensures `.swarmforge/` and `.worktrees/` are in git excludes for target projects; preserve that behavior when touching startup code.

## Config Rules
- `swarmforge/swarmforge.conf` lines are `window <role> <agent> <worktree> [task|batch] [extra-cli-args...]`.
- Supported agents are only `claude`, `codex`, `copilot`, and `grok`; supported receive modes are only `task` and `batch`.
- Role names may not contain underscores, roles must be unique, and non-`master`/`none` worktree names must be unique and path-safe.
- Every configured role must have `swarmforge/roles/<role>.prompt`; the first configured window is the cleanup window.
- `master` and `none` run in the main checkout; other worktrees are created under `.worktrees/<worktree>` on branch `swarmforge-<worktree>`.

## Handoff Protocol Gotchas
- Agents never send tmux messages directly; `handoffd.bb` owns tmux wake-ups and delivers durable inbox files.
- Draft handoffs contain headers only, then go through `swarm_handoff.sh <draft-file>`; the helper generates bodies and removes successful drafts.
- Valid draft types are only `awake`, `git_handoff`, and `note`; `git_handoff` requires a task and an exactly 10-character hex commit abbreviation that resolves to one commit.
- Handoff runtime files under `.swarmforge/handoffs/` are audit/runtime state; do not hand-edit, merge, stage, or commit them.

## Terminal Behavior
- Terminal backend is auto-detected from `SWARMFORGE_TERMINAL`, then macOS Terminal/iTerm via `osascript`, then Windows Terminal, then `none`.
- Backend adapters live in `swarmforge/scripts/terminal-adapters/` and are validated as executable helper scripts at startup.
- The launcher uses a project-specific tmux socket under `/tmp/swarmforge-<uid-or-user>/` and honors tmux `base-index` / `pane-base-index`.
