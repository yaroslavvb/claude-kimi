# claude-kimi

Claude Code sessions whose model is **Kimi K3 on his own Modal endpoint**. The harness is Claude Code itself:
the same TUI, tools, subagents, `/resume`, transcripts and hooks. A Kimi session is a tmux session like his
other Claude sessions, and sessionpipe mirrors it to the spacesheep board like any other. Only the model
behind it changes.

Built Sun 11 Oct 2026 (week 41). Tested on the M5 (Claude Code 2.1.286) and mordast (2.1.296).

## Quick start

```bash
git clone git@github.com:yaroslavvb/claude-kimi.git ~/git0/claude-kimi    # or anywhere
mkdir -p ~/.config/claude-kimi && cp ~/git0/claude-kimi/endpoint.env.example ~/.config/claude-kimi/endpoint.env
chmod 600 ~/.config/claude-kimi/endpoint.env && $EDITOR ~/.config/claude-kimi/endpoint.env   # the Modal key pair
~/git0/claude-kimi/bin/claude-kimi install    # links ~/.local/bin/claude-kimi, writes ~/.claude-kimi-modal
claude-kimi check                             # the endpoint alone: models, one message, one tool call
claude-kimi selftest                          # one headless Claude Code turn: Read, Bash, a subagent
claude-kimi tmux ~/some/folder                # a Kimi session in tmux session `claude-kimi`
( crontab -l; echo "* * * * * $HOME/.local/bin/claude-kimi up >> $HOME/.config/claude-kimi/up.log 2>&1 # claude-kimi keepalive" ) | crontab -
```

The last line is optional. It brings the tmux session back within a minute whenever it is missing, for example
after a tmux restart, starting it in this repo's folder.

`claude-kimi` with no subcommand is `claude` on Kimi, so every claude flag works: `claude-kimi -p "…"`,
`claude-kimi --resume`, `claude-kimi --dangerously-skip-permissions`. `claude-kimi help` prints the full usage.

## How it works

The Modal endpoint is SGLang, and SGLang serves Anthropic's Messages API at `/v1/messages`, with streaming,
thinking blocks and tool use. So Claude Code can talk to it directly, with no translation proxy:

| piece | where | what |
|---|---|---|
| config home | `~/.claude-kimi-modal` | `CLAUDE_CONFIG_DIR` for every Kimi session. Its `settings.json` `env` block carries the endpoint, so anything that starts `claude` on this home talks to Kimi, including sessionpipe's daemon resuming a session from the board. Transcripts live in its `projects/`. |
| endpoint | `~/.config/claude-kimi/endpoint.env` (0600, never in git) | `KIMI_BASE_URL`, `MODAL_KEY`, `MODAL_SECRET`. The Modal proxy-auth pair travels as `ANTHROPIC_CUSTOM_HEADERS`. `ANTHROPIC_AUTH_TOKEN` is a placeholder that Modal ignores, so the home never holds or sends a claude.ai login. |
| tmux | session `claude-kimi` | one window per folder, started in bypass-permissions mode like the lab boxes' sessions (`CLAUDE_KIMI_PERMISSION_MODE` changes it). Named like his account sessions (`claude-<key>`) since Fri 23 Oct 2026; it was `kimi-modal` before. |

Rotate the key by editing `endpoint.env`. The next `claude-kimi` run sees the file is newer than the home's
`settings.json` and rewrites it. Running `claude-kimi setup` does the same by hand.

### Three things Claude Code needs from this endpoint

Each was found by testing, Sun 11 Oct 2026:

1. **Keep Claude's model names.** The CLI rejects a name it does not know before it sends anything:
   `[claude-code:unrecognized_model] {"model":"moonshotai/Kimi-K3"}`. The endpoint serves Kimi K3 whatever
   `model` says, so the status line still reads "Opus 5.5" while Kimi is answering.
2. **Effort must be low, high or max.** Kimi K3 answers 400 to Claude Code's default `medium` and to `xhigh`.
   The home pins `CLAUDE_CODE_EFFORT_LEVEL=high`. Set `CLAUDE_KIMI_EFFORT=max` or `low` to change it, and don't
   pick medium or xhigh in `/effort`. The `effortLevel` key in settings does not override it; the env var does.
3. **A parent's environment beats `settings.json`.** A `claude` started from another Claude session, from the
   desktop app, or from a tmux server one of those started inherits `ANTHROPIC_BASE_URL` and friends. Those win
   over the home's settings, and the requests went to Anthropic with the placeholder token
   (`401 Invalid bearer token`). So the wrapper drops every inherited `ANTHROPIC_*` / `CLAUDE_CODE_*` variable
   and sets its own, and tmux windows run the wrapper, never bare `claude`.

### sessionpipe

`claude-kimi setup` copies the hooks from `~/.claude/settings.json` (sessionpipe's 14 Claude Code events, the
`<ai>` tag guard and the shell guards) into the Kimi home, along with that machine's permission defaults (`allow`, `defaultMode`, `disableAutoMode`).
It leaves out `memory-store-guard.sh` (since Fri 23 Oct 2026): that guard moves memory writes into the accounts'
shared store, and this home keeps its own memory by design. sessionpipe's installer also finds the home by itself (it globs `~/.claude[-_.]*`) and
keeps its hooks current there; `sessionpipe status` lists `~/.claude-kimi-modal/settings.json  current`.
Verified on the M5: a Kimi session's events reach the outbox and the spacesheep sink. A session resumed with
plain `claude` and `CLAUDE_CONFIG_DIR=~/.claude-kimi-modal`, and no wrapper, still runs on Kimi and remembers
its context. That is the path sessionpipe's control daemon takes when he messages a session from the board.

### From the spacesheep board

A Kimi session can be messaged from [spacesheep.dev/sessions](https://spacesheep.dev/sessions) when it runs
on a machine whose sessionpipe control is paired (the M5, the Intel, and mordast since Tue 13 Oct 2026), in a
folder that machine allows (`~/git0` or `~/git` on the Macs; `~/claude-kimi` among mordast's). To start one:

```bash
cd ~/git0/claude-kimi && claude-kimi -p "Kimi session for the board: say hello"
```

Each board message runs `claude -p --permission-mode auto --resume <id>` in that folder, with the session's
own config home. The home's `settings.json` makes that a Kimi turn, and auto mode's permission classifier
runs on Kimi too. Tested Sun 11 Oct 2026 with the daemon's launchd environment: Bash ran, and the resumed
session still had its context. Keep a board session out of an open TUI, so two processes never write one
transcript.

### What doesn't work

- **Remote Control** (`claude remote-control`, the Claude phone app). It refuses with "Remote Control is only
  available with claude.ai subscriptions". Signing this home in to claude.ai would make Claude Code send that
  claude.ai token to the Modal endpoint with every request, so it stays signed out. Reach Kimi sessions
  remotely through the sessionpipe board on a paired machine, or by ssh plus `tmux attach -t '=claude-kimi'`.
- **WebSearch** and other Anthropic server-side tools. WebFetch works: it fetches locally and summarises with Kimi.
- `/model` choices are labels. All of them are Kimi K3.

## Why Claude Code on Kimi, and not an open-source agent

His ask was a Claude-like session that lives in tmux, plugs into sessionpipe, and uses Kimi. The candidates
he looked at, Sun 11 Oct 2026:

- **[OpenCode](https://www.lowcode.agency/blog/claude-code-vs-opencode)** (MIT, model-agnostic TUI). It would
  take the endpoint, but it is a different harness: different sessions, commands and tools, and sessionpipe has
  no adapter for it.
- **[Kimi Code CLI](https://www.codeagentswarm.com/en/guides/how-to-use-kimi-code)** (Moonshot, MIT, Kimi K3
  is its recommended model). It has Claude-Code-like hooks (SessionStart, UserPromptSubmit, PostToolUse, Stop),
  so a sessionpipe adapter is plausible, but none exists yet. It also authenticates against Moonshot's own
  platform, not a Modal endpoint.
- **Codex CLI / Gemini CLI.** sessionpipe supports both, but neither speaks to an Anthropic-format endpoint,
  and neither behaves like Claude.
- **A translation proxy** (claude-code-router, LiteLLM). Not needed:
  [SGLang serves the Messages API natively](https://docs.sglang.io/docs/basic_usage/anthropic_api), as does
  [vLLM](https://docs.vllm.ai/en/v0.14.0/serving/integrations/claude_code/), and SGLang ignores the `model`
  field.

Claude Code on Kimi needs no new harness, no proxy and no sessionpipe change. If the endpoint ever moves to a
server without `/v1/messages`, put a translation proxy in front of it and keep everything else.

## Cost

The live `kimi-k3` endpoint (`ep-mEw0EalWSYkHJ3GB6SrVKk`, created 6 Sep 2026) is **Shared**, billed per token:
$3 per million input tokens, $0.30 cached, $15 completion (`modal endpoint info … --json` reads
`"serving_mode": "shared"`; corrected Tue 13 Oct 2026, when this section still said GPU-seconds). There is no
idle bill. A Claude Code session sends large prompts: the system prompt plus tool definitions come to about
130 KB per request, so the cached-input rate does most of the work. `ep-aRzRWPCdQ0I0gaRxvlp6r6`, also named
`kimi-k3`, is the July endpoint, stopped 6 Sep; its dashboard page still opens but it serves nothing. The
public console's $100 meter does not see these requests: they go to the raw endpoint, not through the console.

## Machines

| machine | installed | tmux |
|---|---|---|
| M5 (`m5-mbp`) | `~/git0/claude-kimi` (Dropbox), `~/.local/bin/claude-kimi` | `kimi-modal`, for testing (the old name, while that session lasts; the next `claude-kimi tmux` opens `claude-kimi`) |
| mordast (nekko.ai) | `~/claude-kimi` (git clone, updated by bundle over ssh) | `claude-kimi`, kept up by the cron keepalive since Fri 23 Oct 2026 |

On mordast, Roman's vault terms bind Kimi sessions like any other agent: no session reads the Esperanto
archive unless he asks for it, and nothing from it leaves for people outside the company.
