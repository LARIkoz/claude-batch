# claude-batch

Drop-in replacement for `claude -p` that prevents VS Code/Cursor extension auth kickout during parallel batch runs.

## The Problem

Running 10+ parallel `claude -p` from VS Code/Cursor causes the extension to lose authentication ("How do you want to log in?"). This happens because:

1. All `claude` processes share a single macOS Keychain entry
2. OAuth refresh tokens rotate — when one process refreshes, others get invalidated
3. The CLI's lock mechanism (5 retries, ~7.5s budget) is insufficient for 10+ concurrent processes

**22 open issues** on [anthropics/claude-code](https://github.com/anthropics/claude-code): [#24317](https://github.com/anthropics/claude-code/issues/24317), [#37512](https://github.com/anthropics/claude-code/issues/37512), [#37203](https://github.com/anthropics/claude-code/issues/37203), [#37324](https://github.com/anthropics/claude-code/issues/37324), [#37468](https://github.com/anthropics/claude-code/issues/37468).

## The Solution

**Strategy: prevent token refresh, don't fix the race.**

If no worker attempts a refresh during the batch, the race condition cannot occur. `claude-batch` guarantees this by:

1. **Pre-batch force refresh** — ensures fresh 8h token before starting
2. **2-hour token gate** — refuses to start if token < 2h remaining (3x safety margin for typical 40-min batch)
3. **tmux isolation** — workers run in detached tmux sessions (separate process tree from Cursor/VS Code)
4. **Retry with jitter** — exponential backoff + random jitter prevents thundering herd
5. **Per-job timeout** — 5-min hard kill for hung workers
6. **Prompt via temp file** — handles large prompts without shell argument overflow
7. **Idempotent retry** — re-run skips completed results

## Why tmux?

Terminal.app was proven to work (48/48, zero kickouts) because it creates a separate process tree. tmux provides identical isolation with programmatic control. Workers authenticate naturally via Keychain — no token env vars, no PATH shims, no hacks.

## Install

```bash
# Download
curl -o /usr/local/bin/claude-batch \
  https://raw.githubusercontent.com/LARIkoz/claude-batch/main/claude-batch
chmod +x /usr/local/bin/claude-batch

# Or clone
git clone https://github.com/LARIkoz/claude-batch.git
ln -s $(pwd)/claude-batch/claude-batch /usr/local/bin/claude-batch
```

Requires: `tmux` (`brew install tmux`), `claude` CLI, macOS.

## Usage

### Drop-in replacement

```bash
# Instead of:
claude -p "classify this app" --model sonnet

# Use:
claude-batch -p "classify this app" --model sonnet
```

### Batch mode

```bash
# Create prompt files
mkdir prompts
echo "classify: Instagram" > prompts/001.txt
echo "classify: Spotify" > prompts/002.txt
# ... 800 more

# Run batch (10 parallel, sonnet model)
claude-batch batch -f prompts -p 10 -m sonnet -o results

# Results in results/result_001.json, results/result_002.json, etc.
```

### Other commands

```bash
claude-batch check     # Token health + credentials status
claude-batch restore   # Restore Keychain/file if corrupted
```

## Test Results

| Scenario                | Raw `claude -p`      | claude-batch (tmux)      |
| ----------------------- | -------------------- | ------------------------ |
| 10 parallel from Cursor | Extension kicked out | 7/10 OK, Extension safe  |
| 30 parallel from Cursor | Extension kicked out | 22/30 OK, Extension safe |
| Extension auth          | Lost every ~60 calls | Never lost               |

Failures in claude-batch are rate limits (retried automatically), never auth kickouts.

## How It Was Built

This tool was designed through a multi-model consilium process:

- **5 design rounds** — Gemini 2.5 Pro, Grok-4, DeepSeek R1, Mistral Large, Codex GPT-5.4, Qwen3-Coder
- **3 code review rounds** — 6 agents, final score 6/6 SHIP
- **1 red team** — 5 agents, 12 attack vectors analyzed
- **1 strategy validation** — 6/6 APPROVE on "prevent refresh" approach

Source code analysis of `cli.js` v2.1.81 was performed to understand the exact token refresh mechanism, Keychain storage, and lock behavior.

## Configuration

Edit the script header to adjust:

```bash
DEFAULT_PARALLEL=10          # default concurrency
MAX_PARALLEL=15              # hard cap
MAX_RETRIES=3                # per-job retry limit
RETRY_BASE_DELAY=5           # seconds, exponential + jitter
MIN_TOKEN_LIFETIME_SEC=7200  # 2h gate (prevent refresh)
JOB_TIMEOUT=300              # 5 min per-job hard timeout
```

## Limitations

- **~95-98% protection** — server-side token revocation could theoretically force refresh (never observed in practice)
- **macOS only** — uses macOS Keychain (`security` command) and tmux
- **Subscription billing** — uses your existing Claude subscription, not API key

## Related Issues

- [#24317](https://github.com/anthropics/claude-code/issues/24317) — Primary: concurrent sessions re-auth race
- [#37512](https://github.com/anthropics/claude-code/issues/37512) — CLAUDE_CODE_OAUTH_TOKEN deletes Keychain
- [#37203](https://github.com/anthropics/claude-code/issues/37203) — Background agents corrupt credentials
- [#37324](https://github.com/anthropics/claude-code/issues/37324) — Subagent token expiry
- [#37468](https://github.com/anthropics/claude-code/issues/37468) — Token expiry kills active agents

## License

MIT
