# sessions-ls

<img width="1392" height="916" alt="image" src="https://github.com/user-attachments/assets/cf8872c6-e989-4a04-afb5-b554e1e6abb5" />

List every terminal coding agent (opencode, claude) running in a tmux pane,
with its session name and status, and jump straight to the pane.

## Install

Needs `tmux` and Python 3 (standard library only, nothing to install). `fzf`
is optional: when present, `/` uses it for search.

```bash
git clone git@github.com:4lb0/sessions-ls.git ~/self/sessions-ls
ln -sf ~/self/sessions-ls/sessions-ls ~/bin/sessions-ls
ln -sf ~/self/sessions-ls/sessions-ls ~/bin/sl
```

## Usage

```
sessions-ls              # interactive picker, refreshes itself every second
sessions-ls | less       # pipeline: plain listing, newest activity first
sessions-ls --plain      # same as above, explicitly
sessions-ls --header     # plain listing with a header row
sessions-ls --json       # machine readable
sl                       # short alias for sessions-ls
```

Interactive keys:

| key            | action                          |
| -------------- | ------------------------------- |
| `↑` `↓` / `j` `k` | move selection               |
| `/`            | search the list                 |
| `Enter`        | jump to that pane in tmux       |
| `g` / `G`      | first / last                    |
| `r`            | re-sort by activity             |
| `q` / `Esc`    | quit                            |

`/` opens [fzf](https://github.com/junegunn/fzf) when it is installed. Without
it, the picker filters as you type: case- and accent-insensitive, so
`sincronizacion` matches `Sincronización`, and multiple words are ANDed.
`Enter` jumps to the highlighted match, `Esc` returns to the full list.

Plain output is shaped like `ls -lht`: long-ish, human ages (`now`, `13m`,
`3h`, `2d`), sorted by most recent activity.

```
STATUS    AGENT     TMUX        ACTIVE  NAME
working  opencode albo:7.0   now     sessions-ls tmux session picker  ~/self/sessions-ls
awaiting claude   pausa:4.1  6h      Sincronización de datos del tablero  ~/work/pausa/all
```

## Statuses

| status     | meaning                                                  |
| ---------- | -------------------------------------------------------- |
| `working`  | the agent is busy (tool running / generating)             |
| `awaiting` | last turn finished, it is waiting for your next message   |
| `question` | the agent asked you a question, the dialog is on screen    |
| `approval` | a permission prompt is on screen, the agent is blocked    |
| `unknown`  | process found but its session could not be read           |

## How it works

* **Panes**: `tmux list-panes -a` gives pane id, pid, title and cwd.
* **Processes**: `ps` finds interactive `opencode` / `claude` processes and
  maps them to a pane through their parent chain (`TMUX_PANE` as a fallback).
* **opencode sessions**: read-only SQLite on
  `~/.local/share/opencode/opencode.db` (`session_v2`). The pane title is
  `OC | <session title>`, so the title (plus the pane cwd) identifies the
  session; `time_busy` / `time_idle` give its status. A session with a running
  subagent is `working` even if its own turn is idle.
* **claude sessions**: transcripts in `~/.claude/projects/<munged-cwd>/*.jsonl`.
  The last `ai-title` entry matches the pane title; the last real transcript
  entry tells whether the turn ended.
* **matching**: a session is claimed by at most one pane. The id passed to
  `opencode -s` / `claude --resume` wins, then the pane title. Panes with no
  title get a session created after their agent process started, so a second
  agent in the same directory does not inherit the first one's name or status.
  An opencode session that has no title yet shows its slug (e.g.
  `playful-wizard`).
* **prompts**: the bottom lines of the pane are checked for permission
  prompts (`approval`) and interactive question dialogs (`question`).

No dependencies beyond the Python 3 standard library.

## tmux binding

Bind it to a popup key, e.g. globally on `C-Space` with a fallback that keeps
Neovim's own `C-Space` mapping intact:

```tmux
bind -n C-Space if-shell -F '#{==:#{pane_current_command},nvim}' \
  'send-keys C-Space' \
  'display-popup -E -b rounded -w 90% -h 60% -T agents ~/bin/sessions-ls'
bind A display-popup -E -b rounded -w 90% -h 60% -T agents ~/bin/sessions-ls
```

`Enter` switches the client to the selected pane, whether it lives in the
current session or another one.

## Options

```
--plain, -l        print and exit
--json             JSON output
--header           header row in plain mode
--watch SECONDS    plain mode, reprint every N seconds
--interval SECONDS interactive refresh interval (default 1)
```

Environment overrides: `SESSION_LS_OPENCODE_DB`, `SESSION_LS_CLAUDE_DIR`.
