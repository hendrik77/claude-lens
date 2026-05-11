# claude-lens

A modular, width-aware status line for [Claude Code](https://claude.ai/code). Shows context window usage, session cost, git branch, model, rate limits, effort mode, and more — auto-compacting when the terminal is narrow.

```
user@host | myproject | main | sonnet | 89k/400k | $1.68 | 51%w | 🔥🤔
```

## Modules

| Module | Full | Compact |
|---|---|---|
| `user_host` | `user@host` | `u@host` |
| `cwd` | `/home/user/dev/myproject` | `myproject` |
| `workspace` | `~/dev/myproject` | `myproject` (project root) |
| `git` | `main*` | `main*` |
| `model` | `claude-sonnet-4-6` | `sonnet` |
| `tokens` | `89k/400k tokens` | `89k/400k` |
| `context_bar` | `▓▓▓░░░░░ 22%` | `22%` |
| `cost` | `~$1.68` | `$1.68` |
| `session_id` | `#d02959` | `#d029` |
| `rate_limit` | `▓▓▓░░░ 51% week` | `51%w` |
| `badges` | `high-effort · thinking` | `🔥🤔` |
| `tasks` | `3 tasks` | `3` |

See [modules.md](modules.md) for the full reference including compact mode options and JSON field names.

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- `jq` (`apt install jq` / `brew install jq`)
- `bash 3+`

## Setup

### 1. Place the skill

If you use [agentskills](https://agentskills.io), clone directly into your skills directory:

```bash
git clone https://github.com/hendrik77/claude-lens ~/.claude/skills/claude-lens
```

### 2. Install the script

```bash
cp ~/.claude/skills/claude-lens/statusline-command.sh ~/.claude/statusline-command.sh
chmod +x ~/.claude/statusline-command.sh
```

### 3. Create your config

```bash
cp ~/.claude/skills/claude-lens/statusline.conf.example ~/.claude/statusline.conf
```

Edit `~/.claude/statusline.conf` to enable or reorder modules.

### 4. Wire into Claude Code

Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline-command.sh"
  }
}
```

## Customisation

Edit `~/.claude/statusline.conf`. The `MODULES` line controls which modules appear and in what order. Any module that can't find its data hides silently.

See [modules.md](modules.md) for all options.

## Adding a new module

Add a `render_<name>_full()` and `render_<name>_compact()` function to `statusline-command.sh`, then add `<name>` to `MODULES` in your conf. The assembly loop picks it up automatically.
