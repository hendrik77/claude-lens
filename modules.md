# Status Line Modules Reference

12 available modules. Enable any combination in `statusline.conf` via the `MODULES=` line.

## Module Table

| Module | Full output | Compact output | JSON field | Status |
|---|---|---|---|---|
| `user_host` | `user@host` | `u@host` | `whoami`/`hostname` | ✓ always |
| `cwd` | `/home/user/dev/myproject` | `myproject` | `.cwd` | ✓ confirmed |
| `git` | `main*` | `main*` | git CLI | ✓ script |
| `model` | `claude-sonnet-4-6` | `sonnet 4.6` | `.model.id` | ✓ confirmed |
| `tokens` | `45k/200k tokens` | `45k/200k` | `.context_window.*` | ✓ confirmed |
| `context_bar` | `▓▓▓▓░░░░ 45%` | `45%` | `.context_window.*` | ✓ confirmed |
| `cost` | `~$1.68` | `$1.68` | `.cost.total_cost_usd` | ✓ confirmed |
| `response_time` | *(hidden)* | *(hidden)* | `.response_time_ms` | ✗ absent |
| `session_id` | `#d0295978` | `#d029` | `.session_id` | ✓ confirmed |
| `permission` | *(hidden)* | *(hidden)* | `.permission_mode` | ✗ absent |
| `compaction` | *(hidden)* | *(hidden)* | `.compacted` | ✗ absent |
| `tasks` | `3 tasks` | `3` | `~/.claude/tasks.json` | ✓ hook |
| `rate_limit` | `▓▓▓░░░ 51.00% week` | `▓▓▓░░░ 51.00%w` | `.rate_limits.*` | ✓ confirmed |
| `badges` | `high-effort · thinking` | `🔥🤔` | `.effort.level`, `.thinking.enabled` | ✓ confirmed |
| `workspace` | `~/dev/myproject` | `myproject` | `.workspace.project_dir` | ✓ confirmed |

Modules marked **✗ absent** hide silently — their fields are not present in the Claude Code statusLine payload (verified against Claude Code 2.1.x).

## Compact Mode Options

### USER_HOST_COMPACT
- `short_host` (default) — `u@host` (first char of user, first 4 of hostname)
- `user_only` — `user`
- `initials` — `u@host` (first char of each)

### CWD_COMPACT
- `basename` (default) — `myproject`
- `tilde` — `~/dev/myproject`
- `short` — `~/d/myproject` (first letter of each intermediate directory)

### MODEL_COMPACT
- `family_version` (default) — `sonnet 4.6`
- `family` — `sonnet`
- `full` — `claude-sonnet-4-6`

### TOKENS_COMPACT
- `numbers` (default) — `45k/200k`
- `percent` — `45%`

### CONTEXT_BAR_COMPACT
- `percent` (default) — `45%`
- `bar` — `▓▓░░` (4-char bar, no label)

## context_bar Color Scheme

- Green: < 50% context used
- Yellow: 50–80%
- Red: > 80%


## tasks Module (optional hook)

The `tasks` module reads `~/.claude/tasks.json` written by a `PreToolUse` hook on `TodoWrite`.

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "TodoWrite",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | jq -c \"{count: ([.todos[] | select(.status != \\\"completed\\\")] | length)}\" > ~/.claude/tasks.json'"
          }
        ]
      }
    ]
  }
}
```

This writes `{"count": N}` with the number of non-completed todos each time the task list is updated. The `tasks` module hides when count is 0 or the file is absent.

## tokens vs context_bar

Both show context usage. Use one or both:
- `tokens` — shows numbers (`45k/200k tokens`), easier to read at a glance
- `context_bar` — shows a visual bar with color warning; good for monitoring fill rate

To use both, include both in MODULES: `MODULES="user_host cwd tokens context_bar ..."`

## RATE_LIMIT_WINDOW

Controls which rate limit window the `rate_limit` module displays:
- `seven_day` (default) — weekly quota usage; resets weekly
- `five_hour` — 5-hour rolling window; useful when you're actively burning quota

Color follows the same green/yellow/red thresholds as `context_bar` (< 50% / 50–80% / > 80%).

## badges module

Shows active mode indicators. Hides entirely when effort is `low` (or absent) and thinking is disabled.

| Condition | Full | Compact |
|---|---|---|
| high effort + thinking | `high-effort · thinking` | `🔥🤔` |
| high effort only | `high-effort` | `🔥` |
| medium effort + thinking | `med-effort · thinking` | `◈🤔` |
| thinking only | `thinking` | `🤔` |

Note: emoji in compact mode are 2 display columns wide but counted as fewer bytes by bash's `${#string}`. Width detection may slightly underestimate line length when badges are in compact mode.

## workspace vs cwd

- `cwd` — tracks wherever you've `cd`'d to inside the session
- `workspace` — always shows the project root (`.workspace.project_dir`)

Use `workspace` instead of `cwd` when you work in subdirectories often and want to see the project name. Use `cwd` when precise location matters.

## cost module — what it shows

The `cost` module displays `.cost.total_cost_usd` — the **cumulative session cost** calculated by Claude Code, not a per-turn estimate. It grows through the conversation.

## Additional payload fields (not yet modules)

These fields exist in the Claude Code statusLine JSON but have no module yet. Useful starting points for custom modules:

| Field | Example value | Notes |
|---|---|---|
| `.session_name` | `verify-statusline-modules` | Human-readable session name |
| `.version` | `2.1.138` | Claude Code version |
| `.fast_mode` | `false` | Whether fast mode is active |
| `.thinking.enabled` | `true` | Whether extended thinking is on |
| `.rate_limits.five_hour.used_percentage` | `0` | 5-hour rate limit usage |
| `.rate_limits.seven_day.used_percentage` | `51` | 7-day rate limit usage |
| `.cost.total_duration_ms` | `1442977` | Total wall-clock session time |
| `.cost.total_api_duration_ms` | `612399` | Total API time in session |
| `.context_window.used_percentage` | `22` | Pre-calculated usage % |
