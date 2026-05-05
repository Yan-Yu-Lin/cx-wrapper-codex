# cx — Clean Codex CLI Wrapper

A lightweight wrapper around [OpenAI Codex CLI](https://github.com/openai/codex) (`codex exec`) that strips the noisy output and gives you just the final message + session ID.

Built for non-interactive use — especially useful when called from another AI agent (like [Claude Code](https://docs.anthropic.com/en/docs/claude-code)) to delegate tasks to Codex.

## Prerequisites

- [OpenAI Codex CLI](https://github.com/openai/codex) installed and configured
- [uv](https://github.com/astral-sh/uv) (the script runs via `uv run --script` — no venv setup needed)

## Install

```bash
# Copy to somewhere on your PATH
cp cx ~/.local/bin/cx
chmod +x ~/.local/bin/cx
```

## Usage

```bash
cx "prompt"                              # Clean output: final message + session ID
cx - < prompt.md                         # Safe long/multiline prompt via stdin
cx --prompt-file prompt.md               # Safe long/multiline prompt via file
cx --full "prompt"                       # Full codex exec passthrough (all noise)
cx resume SESSION_ID "follow-up"         # Continue specific conversation
cx resume --last "follow-up"             # Continue last conversation (interactive only)
cx show SESSION_ID                       # Show full conversation history
cx show --last                           # Show last session history (interactive only)
```

### Flag passthrough

All `codex exec` flags pass through naturally:

```bash
cx --model gpt-5.4-mini "quick task"     # Different model
cx --sandbox read-only "investigate X"   # Read-only sandbox
cx -C /path/to/repo "fix the build"     # Different working directory
cx -i screenshot.png "fix this UI bug"  # Attach image
```

### Progress output

Clean mode streams lightweight progress to stderr while Codex is running, then prints the final response to stdout when the turn completes. This keeps automation-friendly final output while avoiding long silent hangs. If Codex emits no events for 2 minutes, cx prints a single warning that it may be waiting on a tool or subprocess.

```bash
cx "prompt"                # Default: timestamped progress on stderr
cx --quiet-progress "..."  # Only start/session/finish progress
cx --no-progress "..."     # Disable progress output
cx --debug-json "..."      # Echo raw codex JSONL events to stderr
```

For long prompts, prefer stdin or `--prompt-file` so shell quoting does not break on quotes, backslashes, dollar signs, or multiline text.

### Multi-turn conversations

Use `resume` for back-and-forth with Codex. The session ID is printed after every response.

```bash
cx "investigate why the auth test is flaky"       # Start — note the session ID
cx resume 019d6dbe-b90f-... "now fix it"           # Follow up with ID
cx resume 019d6dbe-b90f-... "run the tests to verify"  # Continue
cx show 019d6dbe-b90f-...                          # Review full conversation
```

> **Note:** `--last` is blocked in non-interactive mode (e.g. when called from Claude Code's Bash tool) because multiple sessions may run concurrently — `--last` could pick the wrong one. Always use explicit session IDs in automation.

## How it works

- **Clean mode (default):** Runs `codex exec --json -o tmpfile`, streams lightweight progress from JSONL events to stderr, reads tmpfile for final message, prints final output + session ID to stdout, deletes tmpfile.
- **Full mode (`--full`):** Raw passthrough to `codex exec`, no filtering.
- **Show:** Parses session JSONL files from `~/.codex/sessions/` — purely local, no Codex call.

The script injects `-s workspace-write` and `--skip-git-repo-check` so it runs non-interactively with workspace-write sandboxing.

---

## Using cx with Claude Code

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), you can teach it how to use `cx` by adding the following to your `~/.claude/CLAUDE.md` (global instructions):

<details>
<summary>Click to expand CLAUDE.md instructions</summary>

````markdown
## cx — Codex CLI Wrapper (use this, not the MCP)

`cx` is a clean wrapper around `codex exec` (OpenAI's Codex agent). It strips the noisy output and gives you just the final message + session ID. Use it to delegate tasks to Codex as a second AI agent.

**Always use `cx` (via Bash) instead of the Codex MCP tools (`mcp__codex-native__codex` / `codex-reply`).** The MCP adds no value over `cx` — both give clean output, but `cx` has more features (show, resume, flag control, full mode). Don't use the MCP server for Codex. Period.

When the user says "use Codex", "ask Codex", "delegate to Codex", or anything about calling Codex — use `cx`.

Located at `~/.local/bin/cx`. Always runs non-interactively (`-s workspace-write --skip-git-repo-check` injected automatically).

### Usage

```bash
cx "prompt"                              # Clean output: final message + session ID
cx - < prompt.md                         # Safe long/multiline prompt via stdin
cx --prompt-file prompt.md               # Safe long/multiline prompt via file
cx --full "prompt"                       # Full codex exec passthrough (all noise)
cx resume --last "follow-up"             # Continue last conversation
cx resume SESSION_ID "follow-up"         # Continue specific conversation
cx show --last                           # Show full conversation history of last session
cx show SESSION_ID                       # Show specific session history
```

### Flag passthrough

All codex exec flags pass through naturally:

```bash
cx --model gpt-5.4-mini "quick task"     # Different model
cx --sandbox read-only "investigate X"   # Read-only sandbox
cx -C /path/to/repo "fix the build"     # Different working directory
cx -i screenshot.png "fix this UI bug"  # Attach image
```

### How it works

- **Clean mode (default)**: Runs `codex exec --json -o tmpfile`, streams lightweight progress from JSONL events to stderr, reads tmpfile for final message, prints final output + session ID to stdout, deletes tmpfile.
- **Full mode (`--full`)**: Raw passthrough to `codex exec`, no filtering.
- **Show**: Parses session JSONL files from `~/.codex/sessions/` — purely local, no Codex call.

### Multi-turn conversations

Use `resume` for back-and-forth with Codex. The session ID is printed after every response.

```bash
cx "investigate why the auth test is flaky"                          # Start — note the session ID
cx resume 019d6dbe-b90f-... "now fix it"                            # Follow up with ID
cx resume 019d6dbe-b90f-... "run the tests to verify"               # Continue
cx show 019d6dbe-b90f-...                                           # Review full conversation
```

**Always use explicit session IDs when resuming.** `--last` is blocked in non-interactive mode (like Claude Code's Bash tool) because multiple sessions may be running Codex simultaneously — `--last` could pick the wrong one. The session ID is printed after every `cx` response.
````

</details>

## License

MIT
