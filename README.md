# delegate-to-opencode

A Claude Code skill that delegates tasks to [opencode](https://opencode.ai) — the open-source AI coding agent — non-interactively, then verifies the result and retries until the task is genuinely complete.

## What it does

When you ask Claude Code to do something "using opencode" / "pakai opencode", this skill:

1. Translates your request into a self-contained `opencode run` call
2. Executes it with `--dangerously-skip-permissions` so it never hangs waiting for approval
3. **Verifies the result** — reads changed files, diffs, or test output to confirm the task is actually done
4. **Retries via `--continue`** if the result is incomplete or wrong, with a specific explanation of what's still missing
5. Reports what changed and which files were affected

## Requirements

- [Claude Code](https://claude.ai/code) installed
- [opencode](https://opencode.ai) installed and configured with a provider/API key (`opencode providers`)

## Installation

### Option A — copy the skill folder

```bash
cp -r . ~/.claude/skills/delegate-to-opencode
```

### Option B — clone directly into your skills folder

```bash
git clone https://github.com/kabesma/delegate-to-opencode ~/.claude/skills/delegate-to-opencode
```

Restart Claude Code (or start a new session) for the skill to appear in the available skills list.

## Usage

Just mention **opencode** as the executor in your request:

```
suruh opencode bikin fungsi validasi email di utils
have opencode fix the failing tests
lewat opencode, refactor the auth middleware
tell opencode to explain the auth flow in this repo
lanjutkan task opencode tadi, tambahin unit testnya
```

Claude Code will invoke this skill automatically — no slash command needed.

## Verified user preferences (pre-configured)

| Setting | Value | Reason |
|---|---|---|
| `--dangerously-skip-permissions` | always on | Prevents the bash process from hanging while waiting for approval |
| Confirmation before run | off | Run immediately once the task is clear |
| Default working directory | context-aware, falls back to project root | Runs in the directory most relevant to the task |

## How the retry loop works

```
opencode run "task" → verify result
        ↓ incomplete/wrong
opencode run --continue "here's what's still wrong: ..."
        ↓ verify again
repeat (max ~3 rounds) → report to user
```

Claude reads the actual file changes or output — not just opencode's "done" message — before deciding whether to retry.
