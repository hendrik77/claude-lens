---
name: claude-lens
description: Use when setting up or recreating the Claude Code status line with modular PS1-style display (user@host, cwd, git, tokens, model, cost, and more) on a new machine or remote VM
---

# Status Line: Modular PS1

## Overview

Configures Claude Code's `statusLine` to display a modular, width-aware status bar. Modules auto-compact on narrow terminals; a second-line fallback handles very narrow widths. Active modules and layout are configured in `~/.claude/statusline.conf`.

## Requirements

- `jq` on PATH (`apt install jq` / `brew install jq`)
- `bash 3+`

## Setup

### 1. Write the script

Create `~/.claude/statusline-command.sh` and `chmod +x` it.  
Full content: see the script block below.

### 2. Write the default conf

Create `~/.claude/statusline.conf`:

```bash
# Claude Code status line configuration
# Ordered list of active modules (space-separated, use _ not -)
# workspace is an alternative to cwd — shows project root instead of current dir
MODULES="user_host cwd git model tokens cost session_id rate_limit badges tasks"

# Layout
MAX_WIDTH=120          # fallback if $COLUMNS unavailable
SEPARATOR=" | "        # rendered between modules
SECOND_LINE_BREAK=3    # wrap after module N onto line 2 (0 = disabled, grow instead)

# Per-module compact overrides
USER_HOST_COMPACT=short_host   # short_host | user_only | initials
CWD_COMPACT=basename           # basename | tilde | short (~/D/p)
MODEL_COMPACT=family_version   # family_version (sonnet 4.6) | family (sonnet) | full (claude-sonnet-4-6)
CONTEXT_BAR_COMPACT=percent    # percent (45%) | bar (▓▓░░ no label)
TOKENS_COMPACT=numbers         # numbers (12k/200k) | percent (45%)
RATE_LIMIT_WINDOW=seven_day    # seven_day | five_hour

# cost module uses .cost.total_cost_usd from Claude Code payload (no manual pricing needed)
```

### 3. Wire it in settings.json

Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline-command.sh"
  }
}
```

Preserve any existing keys — only add/update the `statusLine` block.

## Customisation

See `modules.md` in this directory for the full 12-module reference, JSON field names, full/compact output examples, and the optional `tasks` hook setup.

## Common Mistakes

- **`command not found: jq`** — install jq
- **Status line not showing** — verify `settings.json` uses `bash ~/.claude/...` (bare `~/.claude/...` may not expand `~`)
- **Script not executable** — run `chmod +x ~/.claude/statusline-command.sh`
- **Module missing** — field may not exist in this Claude Code version; modules hide silently when their JSON field is absent

---

## Script: ~/.claude/statusline-command.sh

```bash
#!/usr/bin/env bash
# shellcheck disable=SC1090
source ~/.claude/statusline.conf 2>/dev/null

MODULES="${MODULES:-user_host cwd tokens}"
SEPARATOR="${SEPARATOR:- | }"
MAX_WIDTH="${MAX_WIDTH:-120}"
SECOND_LINE_BREAK="${SECOND_LINE_BREAK:-0}"

INPUT=$(cat)
CWD=$(echo "$INPUT"         | jq -r '.cwd // empty')
TOKENS_USED=$(echo "$INPUT" | jq -r '.context_window.total_input_tokens // empty')
TOKENS_MAX=$(echo "$INPUT"  | jq -r '.context_window.context_window_size // empty')
MODEL=$(echo "$INPUT"       | jq -r '.model.id // empty')
COST_USD=$(echo "$INPUT"       | jq -r '.cost.total_cost_usd // empty')
SESSION=$(echo "$INPUT"        | jq -r '.session_id // empty')
RATE_7D=$(echo "$INPUT"        | jq -r '.rate_limits.seven_day.used_percentage // empty')
RATE_5H=$(echo "$INPUT"        | jq -r '.rate_limits.five_hour.used_percentage // empty')
EFFORT=$(echo "$INPUT"         | jq -r '.effort.level // empty')
THINKING=$(echo "$INPUT"       | jq -r '.thinking.enabled // empty')
WORKSPACE_DIR=$(echo "$INPUT"  | jq -r '.workspace.project_dir // empty')

C_RESET='\033[00m'
C_USER='\033[01;32m'
C_CWD='\033[01;34m'
C_TOKEN='\033[00;33m'
C_GREEN='\033[0;32m'
C_YELLOW='\033[0;33m'
C_RED='\033[0;31m'

ESC=$(printf '\033')
strip_ansi() { printf '%s' "$1" | sed "s/${ESC}\[[0-9;]*m//g"; }

render_user_host_full()    { printf "${C_USER}%s@%s${C_RESET}" "$(whoami)" "$(hostname -s)"; }
render_user_host_compact() {
  case "${USER_HOST_COMPACT:-short_host}" in
    user_only) printf "${C_USER}%s${C_RESET}" "$(whoami)" ;;
    initials)  printf "${C_USER}%s@%s${C_RESET}" "$(whoami | cut -c1)" "$(hostname -s | cut -c1-3)" ;;
    *)         printf "${C_USER}%s@%s${C_RESET}" "$(whoami | cut -c1)" "$(hostname -s | cut -c1-4)" ;;
  esac
}

render_cwd_full() {
  [[ -z "$CWD" ]] && return
  printf "${C_CWD}%s${C_RESET}" "$CWD"
}
render_cwd_compact() {
  [[ -z "$CWD" ]] && return
  case "${CWD_COMPACT:-basename}" in
    tilde) printf "${C_CWD}%s${C_RESET}" "${CWD/#$HOME/\~}" ;;
    short)
      local rel="${CWD/#$HOME/\~}"
      local base; base=$(basename "$rel")
      local dir;  dir=$(dirname "$rel")
      local short_dir
      short_dir=$(printf '%s' "$dir" | awk -F'/' '{
        printf "%s", $1
        for (i=2; i<=NF; i++) printf "/%s", substr($i,1,1)
        print ""
      }')
      printf "${C_CWD}%s/%s${C_RESET}" "$short_dir" "$base"
      ;;
    *) printf "${C_CWD}%s${C_RESET}" "$(basename "$CWD")" ;;
  esac
}

render_git_full() {
  [[ -z "$CWD" ]] && return
  local branch
  branch=$(git -C "$CWD" rev-parse --abbrev-ref HEAD 2>/dev/null) || return
  local dirty
  dirty=$(git -C "$CWD" status --porcelain 2>/dev/null | head -1)
  [[ -n "$dirty" ]] && branch+="*"
  printf "%s" "$branch"
}
render_git_compact() { render_git_full; }

render_model_full() {
  [[ -z "$MODEL" ]] && return
  printf "%s" "$MODEL"
}
render_model_compact() {
  [[ -z "$MODEL" ]] && return
  case "${MODEL_COMPACT:-family_version}" in
    full) printf "%s" "$MODEL" ;;
    family_version)
      local family version
      family=$(printf '%s' "$MODEL" | sed 's/claude-\([a-z]*\)-.*/\1/')
      if [[ -z "$family" || "$family" == "$MODEL" ]]; then
        printf "%s" "$MODEL"
      else
        version=$(printf '%s' "$MODEL" | sed "s/claude-${family}-//" | tr '-' '.')
        printf "%s %s" "$family" "$version"
      fi
      ;;
    *)
      local family
      family=$(printf '%s' "$MODEL" | sed 's/claude-\([a-z]*\)-.*/\1/')
      printf "%s" "$family"
      ;;
  esac
}

render_tokens_full() {
  [[ -z "$TOKENS_USED" || -z "$TOKENS_MAX" ]] && return
  local used_k total_k
  used_k=$(awk "BEGIN {printf \"%.0f\", $TOKENS_USED / 1000}")
  total_k=$(awk "BEGIN {printf \"%.0f\", $TOKENS_MAX / 1000}")
  printf "${C_TOKEN}%sk/%sk tokens${C_RESET}" "$used_k" "$total_k"
}
render_tokens_compact() {
  [[ -z "$TOKENS_USED" || -z "$TOKENS_MAX" ]] && return
  case "${TOKENS_COMPACT:-numbers}" in
    percent)
      local pct
      pct=$(awk "BEGIN {printf \"%.0f\", ($TOKENS_USED / $TOKENS_MAX) * 100}")
      printf "${C_TOKEN}%s%%${C_RESET}" "$pct"
      ;;
    *)
      local used_k total_k
      used_k=$(awk "BEGIN {printf \"%.0f\", $TOKENS_USED / 1000}")
      total_k=$(awk "BEGIN {printf \"%.0f\", $TOKENS_MAX / 1000}")
      printf "${C_TOKEN}%sk/%sk${C_RESET}" "$used_k" "$total_k"
      ;;
  esac
}

_bar_color() {
  local pct=$1
  if   (( pct < 50 )); then printf '%s' "$C_GREEN"
  elif (( pct < 80 )); then printf '%s' "$C_YELLOW"
  else                      printf '%s' "$C_RED"
  fi
}
_bar_chars() {
  local pct=$1 width=$2
  local filled empty i bar=""
  filled=$(awk "BEGIN {n=int(($pct/100.0)*$width+0.5); print (n>$width)?$width:n}")
  empty=$(( width - filled ))
  for ((i=0; i<filled; i++)); do bar+="▓"; done
  for ((i=0; i<empty;  i++)); do bar+="░"; done
  printf '%s' "$bar"
}
render_context_bar_full() {
  [[ -z "$TOKENS_USED" || -z "$TOKENS_MAX" ]] && return
  local pct color bar
  pct=$(awk "BEGIN {printf \"%.0f\", ($TOKENS_USED / $TOKENS_MAX) * 100}")
  color=$(_bar_color "$pct")
  bar=$(_bar_chars "$pct" 8)
  printf "${color}%s %s%%${C_RESET}" "$bar" "$pct"
}
render_context_bar_compact() {
  [[ -z "$TOKENS_USED" || -z "$TOKENS_MAX" ]] && return
  local pct color
  pct=$(awk "BEGIN {printf \"%.0f\", ($TOKENS_USED / $TOKENS_MAX) * 100}")
  color=$(_bar_color "$pct")
  case "${CONTEXT_BAR_COMPACT:-percent}" in
    bar) printf "${color}%s${C_RESET}" "$(_bar_chars "$pct" 4)" ;;
    *)   printf "${color}%s%%${C_RESET}" "$pct" ;;
  esac
}

# .cost.total_cost_usd is pre-calculated by Claude Code (cumulative session cost)
render_cost_full()    { [[ -z "$COST_USD" ]] && return; printf "~\$%.2f" "$COST_USD"; }
render_cost_compact() { [[ -z "$COST_USD" ]] && return; printf "\$%.2f" "$COST_USD"; }

# .response_time_ms absent from payload — always hides
render_response_time_full()    { return; }
render_response_time_compact() { return; }

render_session_id_full()    { [[ -z "$SESSION" ]] && return; printf "#%s" "${SESSION:0:6}"; }
render_session_id_compact() { [[ -z "$SESSION" ]] && return; printf "#%s" "${SESSION:0:4}"; }

# .permission_mode absent from payload — always hides
render_permission_full()    { return; }
render_permission_compact() { return; }

# .compacted absent from payload — always hides
render_compaction_full()    { return; }
render_compaction_compact() { return; }

render_tasks_full() {
  local f="$HOME/.claude/tasks.json"
  [[ ! -f "$f" ]] && return
  local count; count=$(jq -r '.count // empty' "$f" 2>/dev/null)
  [[ -z "$count" || "$count" == "0" ]] && return
  printf "%d tasks" "$count"
}
render_tasks_compact() {
  local f="$HOME/.claude/tasks.json"
  [[ ! -f "$f" ]] && return
  local count; count=$(jq -r '.count // empty' "$f" 2>/dev/null)
  [[ -z "$count" || "$count" == "0" ]] && return
  printf "%d" "$count"
}

_rate_pct() {
  case "${RATE_LIMIT_WINDOW:-seven_day}" in
    five_hour) printf '%s' "$RATE_5H" ;;
    *)         printf '%s' "$RATE_7D" ;;
  esac
}
_rate_label() {
  case "${RATE_LIMIT_WINDOW:-seven_day}" in
    five_hour) printf '5h' ;;
    *)         printf 'week' ;;
  esac
}
render_rate_limit_full() {
  local pct; pct=$(_rate_pct)
  [[ -z "$pct" ]] && return
  local pct_int; pct_int=$(awk "BEGIN {printf \"%.0f\", $pct}")
  local pct_fmt; pct_fmt=$(awk "BEGIN {printf \"%.2f\", $pct}")
  local color; color=$(_bar_color "$pct_int")
  local bar;   bar=$(_bar_chars "$pct_int" 6)
  printf "${color}%s %s%% %s${C_RESET}" "$bar" "$pct_fmt" "$(_rate_label)"
}
render_rate_limit_compact() {
  local pct; pct=$(_rate_pct)
  [[ -z "$pct" ]] && return
  local pct_int; pct_int=$(awk "BEGIN {printf \"%.0f\", $pct}")
  local pct_fmt; pct_fmt=$(awk "BEGIN {printf \"%.2f\", $pct}")
  local color; color=$(_bar_color "$pct_int")
  local bar;   bar=$(_bar_chars "$pct_int" 6)
  printf "${color}%s %s%%w${C_RESET}" "$bar" "$pct_fmt"
}

render_badges_full() {
  local parts=() result="" sep=""
  case "$EFFORT" in
    high)   parts+=("high-effort") ;;
    medium) parts+=("med-effort") ;;
  esac
  [[ "$THINKING" == "true" ]] && parts+=("thinking")
  [[ ${#parts[@]} -eq 0 ]] && return
  for p in "${parts[@]}"; do result+="${sep}${p}"; sep=" · "; done
  printf "%s" "$result"
}
render_badges_compact() {
  local out=""
  case "$EFFORT" in
    high)   out+="🔥" ;;
    medium) out+="◈" ;;
  esac
  [[ "$THINKING" == "true" ]] && out+="🤔"
  [[ -z "$out" ]] && return
  printf "%s" "$out"
}

render_workspace_full() {
  [[ -z "$WORKSPACE_DIR" ]] && return
  printf "${C_CWD}%s${C_RESET}" "${WORKSPACE_DIR/#$HOME/\~}"
}
render_workspace_compact() {
  [[ -z "$WORKSPACE_DIR" ]] && return
  printf "${C_CWD}%s${C_RESET}" "$(basename "$WORKSPACE_DIR")"
}

assemble() {
  local mode=$1
  local parts=() mod val result="" sep=""
  for mod in ${MODULES}; do
    [[ "$mod" =~ ^[a-z_]+$ ]] || continue
    val=$(render_${mod}_${mode} 2>/dev/null)
    [[ -n "$val" ]] && parts+=("$val")
  done
  for val in "${parts[@]}"; do
    result+="${sep}${val}"
    sep="${SEPARATOR}"
  done
  printf '%s' "$result"
}

WIDTH=$(tput cols </dev/tty 2>/dev/null || printf '%s' "${COLUMNS:-${MAX_WIDTH}}")
LINE=$(assemble full)
LINE_PLAIN=$(strip_ansi "$LINE")

if (( ${#LINE_PLAIN} > WIDTH )); then
  LINE=$(assemble compact)
  LINE_PLAIN=$(strip_ansi "$LINE")
fi

if [[ "${SECOND_LINE_BREAK}" -gt 0 ]] && (( ${#LINE_PLAIN} > WIDTH )); then
  read -ra all_mods <<< "$MODULES"
  L1="${all_mods[*]:0:$SECOND_LINE_BREAK}"
  L2="${all_mods[*]:$SECOND_LINE_BREAK}"
  LINE1=$(MODULES="$L1" assemble compact)
  LINE2=$(MODULES="$L2" assemble compact)
  printf '%s\n%s' "$LINE1" "$LINE2"
  exit 0
fi

printf '%s' "$LINE"
```
