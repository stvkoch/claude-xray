# claude-xray

Live X-ray of a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) session in your terminal, built with [Textual](https://github.com/Textualize/textual).

It tails the transcript Claude Code writes to `~/.claude/projects/` and shows, refreshing every second:

- **Context**: current size, peak, total output, compactions, with a per-response history and sparkline
  - **Inside the context** (`d` in the context dialog, or Enter on a row of its history): what was actually sent to the model, rebuilt from the request log Claude Code writes. Shows the system prompt, every tool definition, and each reminder, prompt, reply, thinking block, tool call and tool result, broken down by category and sorted by size. Enter opens any item in full. The token split is estimated from character counts and scaled to the real total the API reported
- **Agents**: subagents launched, running and finished, with their own context size and tools
  - **Focus mode** (`Enter` or `f`): follow one subagent live, like `tail -f`: its brief, thinking, text, tool calls with results, and the report it hands back
  - **Brief vs result** (`b`): what the parent asked next to what came back, plus a guess at whether the parent used it. The guess takes the report's distinctive terms that weren't already in the brief (paths, identifiers, `code`, long words) and checks how many show up in what the main thread wrote between the hand-back and the next user prompt. It's a heuristic, not proof: a low score marks an agent worth a look, not a wasted one
- **Skills**: invoked (Skill tool or `/command`), available, newly added, and which skill files were read
- **Activity**: every tool call (main thread and subagents), with full input and result

It is read-only: it never writes to Claude's files.

## Install

```sh
git clone https://github.com/stvkoch/claude-xray.git ~/.local/share/claude-xray
cd ~/.local/share/claude-xray
python3 -m venv venv && venv/bin/pip install -r requirements.txt
cat > ~/.local/bin/claude-xray <<'SH'
#!/bin/sh
exec "$HOME/.local/share/claude-xray/venv/bin/python" "$HOME/.local/share/claude-xray/claude-xray" "$@"
SH
chmod +x ~/.local/bin/claude-xray
```
<img height="500" alt="image" src="https://github.com/user-attachments/assets/0a627e65-aa92-4691-96d9-4fbfcc9f94c9" />
<img height="500" alt="image" src="https://github.com/user-attachments/assets/eb158389-a43a-409f-b710-8df6cbdefca3" />
<img height="500" alt="image" src="https://github.com/user-attachments/assets/3799e1ab-b115-4a93-a224-4320f72fd7b5" />


## Usage

Run it in a second terminal, from the same directory as your Claude Code session:

```sh
claude-xray                 # pick a session from a list
claude-xray --latest        # follow the newest session, switching after /clear
claude-xray --all           # list/follow sessions of every project
claude-xray --session ID    # open a specific session id
claude-xray --once          # print a plain-text snapshot and exit
claude-xray --headless      # also list headless `claude -p` / SDK runs (hidden by default)
```

| Key | Action |
| --- | --- |
| `1` `2` `3` | focus Agents, Skills, Activity |
| `↑` `↓` `Enter` | navigate and open the details dialog (focus mode on an agent) |
| `f` | focus mode: the selected agent, or the newest running one |
| `b` | brief vs result: the selected agent, or the newest finished one |
| `[` `]` | previous / next agent (inside focus and brief vs result) |
| `v` | full output instead of compact (inside focus) |
| `c` | context details (`d` inside it: what's in the context) |
| `s` | switch session |
| `a` / `h` | in the session list: all projects / show headless `claude -p` runs |
| `p` | pause refresh |
| `Esc` | close dialog |
| `q` | quit |

Claude Code only creates a new session file when the first message is sent, so after `/clear` the new session shows up once you type something.
