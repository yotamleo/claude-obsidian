# claude-obsidian Hooks

Plugin hooks for the claude-obsidian wiki vault. All hooks are defined in `hooks.json` and dispatched through a single Python script: `scripts/wiki-hooks.py`. The dispatcher resolves the vault on its own, so the hooks behave the same whether the plugin lives inside the vault, is installed globally, or is added to a separate Obsidian vault.

## Events

| Event | Type | Subcommand | Purpose |
|---|---|---|---|
| `SessionStart` | command | `session-start` | Loads `wiki/hot.md` into context via the dispatcher. A second command hook reaps stale wiki locks (`wiki-lock.sh clear-stale`). Matcher: startup / resume. (Fork notes: prompt-type hooks are removed in this fork — unsupported by our Claude Code baseline. The clear-stale hook uses a cwd-relative path, so it only runs in clone-as-vault setups; elsewhere it silently no-ops.) |
| `PostCompact` | command | `post-compact` | Re-loads `wiki/hot.md` after context compaction. Hook-injected context does not survive compaction (only `CLAUDE.md` does), so this hook restores the hot cache mid-session. |
| `PostToolUse` | command | (fork shell hook) | Auto-commits wiki changes after `Write` or `Edit`. This fork keeps its lock-aware shell hook (defers auto-commit while wiki locks are held; `.vault-meta/auto-commit.disabled` kill-switch) instead of the dispatcher subcommand — `post-tool-use` exists in the dispatcher but is not registered. |
| `Stop` | command | `stop` | If `git diff --name-only HEAD` shows any `wiki/` change, prints the `WIKI_CHANGED:` marker on stdout to ask Claude to refresh `wiki/hot.md`. |

## Vault Resolution

Each subcommand resolves the vault using the chain below. The first hit wins; every step is fail-soft, so a malformed or missing input simply falls through to the next step.

1. `$CLAUDE_OBSIDIAN_VAULT` — explicit override.
2. The `Path:` line under `## Wiki Knowledge Base` in `$CLAUDE_PROJECT_DIR/CLAUDE.md`. Claude Code sets `CLAUDE_PROJECT_DIR` for hook execution.
3. The `Path:` line under `## Wiki Knowledge Base` in `./CLAUDE.md` (current working directory).
4. Walk up from the cwd looking for a directory that contains `.obsidian/` (the standard Obsidian vault marker) or both `wiki/` and `.vault-meta/`. Capped at `$HOME` and at most 6 levels.
5. `~/.claude/claude-obsidian.json` — user-level config: `{"version": 1, "vault": "/abs/path"}`.

If nothing resolves, the dispatcher exits 0 silently. A hook never breaks a Claude Code session.

### Off-directory vaults

If your vault is not the cwd of the project where you run Claude Code, point the dispatcher at it once:

```sh
scripts/wiki-hooks.py init --vault /abs/path/to/vault
```

That writes `~/.claude/claude-obsidian.json`, after which every subcommand resolves the vault regardless of cwd. You can also just export `CLAUDE_OBSIDIAN_VAULT` for one-shot overrides.

## Failure & logging

Vault-scoped log lines go to `<vault>/.vault-meta/hooks.log`; lines logged before a vault resolves go to `<plugin-root>/logs/wiki-hooks.log`. Logs truncate to roughly the last 500 KB once they exceed 1 MB. The dispatcher always returns exit code 0. (Fork note: upstream #27 also shipped a `debug-env.sh` hook that dumps the full environment to a log file; we drop it — env dumps can capture secrets.)

## Known Issue: Plugin Hooks STDOUT Bug

`anthropics/claude-code#10875` documents that **plugin hook STDOUT may not be captured** by Claude Code, while identical inline hooks in `settings.json` work correctly.

**Impact**: If this bug is active in your Claude Code version, the command-type SessionStart and PostCompact hooks may not inject context as expected.

**Workaround**: This fork does not register prompt-type hooks (unsupported by our Claude Code baseline). If the STDOUT bug is active in your Claude Code version, copy the SessionStart hook config from `hooks.json` into your user-level `~/.claude/settings.json` — inline settings hooks are not affected by the plugin-stdout bug.

**Test for the bug**: After installing the plugin, open a fresh Claude Code session in a directory containing a populated `wiki/hot.md`. Ask Claude "what's in the hot cache?". If Claude has no idea, the STDOUT bug is active in your version.

## Non-Vault Sessions

The dispatcher exits 0 with no output when no vault resolves, so the plugin is safe to install globally without breaking non-vault Claude Code sessions.
