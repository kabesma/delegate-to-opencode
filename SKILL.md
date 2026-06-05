---
name: delegate-to-opencode
description: Delegate a task to opencode (the AI coding agent CLI from opencode.ai) non-interactively via `opencode run`. Use this skill whenever the user asks to get something done "using opencode", "with opencode", "via opencode", "tell opencode to", "have opencode", "run it in opencode", "pakai opencode", "lewat opencode", "suruh opencode" — i.e. any time the user names opencode as the executor of a task (building a feature, fixing a bug, refactoring, writing/editing files, or answering questions about a codebase). Also trigger for "continue the previous opencode task" or resuming an earlier opencode session. Do NOT use this skill to configure opencode (provider/API key/model), to view usage/cost stats, or when the user is only asking about opencode without requesting a task run — handle those with opencode commands directly. If the user asks to do a task WITHOUT mentioning opencode, just do it yourself as usual; do not trigger this skill.
---

# OpenCode Task Runner

This skill wraps delegating a task to **opencode** — the AI coding agent from opencode.ai installed on the user's machine. The idea is simple: the user says "do X using opencode", you translate that into a non-interactive `opencode run` call, execute it, then report the result honestly.

opencode is a separate agent with its own LLM provider. It can read, write, and modify files in the working directory and run commands. So invoking opencode means handing real execution to another agent — not just a read-only query. Treat its output as third-party work you must verify and summarize, not swallow whole.

## Confirmed user preferences

These three decisions are already confirmed by the user — apply them as defaults, don't re-ask each time:

1. **Auto-approve permissions.** Always include `--dangerously-skip-permissions`. Without it, opencode stalls waiting for approval and the bash process hangs until timeout. The user is aware of the implications (opencode is free to edit files and run commands).
2. **Run immediately.** As soon as the user asks to "do X using opencode", execute right away — no need to echo the command and wait for extra confirmation.
3. **Directory follows context.** Run in the directory relevant to the task/conversation at hand. If unclear, default to `/Users/ilham/atlas`. Pass it via `--dir`.

## Workflow

### 1. Compose the task message
Take the user's intent and turn it into one clear, self-contained instruction for opencode — opencode does not see your conversation, so the message must stand on its own. If the user gave important context (file names, error messages, constraints, a definition of "done"), fold it into the message. Don't add assumptions that weren't requested.

**Don't pre-solve.** The message should describe the problem or goal — not your analysis or solution. Include context but NOT your diagnosis, root cause, or proposed fix. opencode is a full AI agent; let it reason and solve. If you find yourself writing "the cause is X, fix it by doing Y", stop — that's you solving it, not delegating it.

### 2. Determine the directory
Pick the working directory based on task context. Default to `/Users/ilham/atlas` when unclear. Pass it via `--dir <path>`.

### 3. Run opencode

Base form:

```bash
opencode run --dangerously-skip-permissions --dir <DIR> "<TASK MESSAGE>"
```

Use the default output format (already nicely formatted and readable). Don't use `--format json` unless the user wants raw data or you need to parse events programmatically.

**On duration & timeout.** An opencode task can take anywhere from a dozen seconds to several minutes. Choose how to run it based on expectations:
- **Light/fast tasks** (codebase questions, small edits): run in the foreground with a generous timeout, e.g. the Bash `timeout` at 600000 ms (10 minutes).
- **Heavy/uncertain tasks** (large features, broad refactors): run with `run_in_background: true` so it doesn't block, then monitor until it finishes. Once done, read the full output before reporting.

When in doubt, start in the foreground; if it looks like it'll run long, switch to background.

### 4. Continue a session (optional)
If the user asks to continue/iterate on a previous opencode task ("continue", "fix what you just did", "keep going"), use `--continue` to resume the last session, or `--session <id>` for a specific one:

```bash
opencode run --continue --dangerously-skip-permissions --dir <DIR> "<FOLLOW-UP INSTRUCTION>"
```

### 5. Verify the result — YOU decide whether it's done
This is the heart of the skill: **do not trust opencode's claims at face value.** You are the one who decides whether the task is actually complete, not opencode.

After opencode finishes:
- Read its output. Note what opencode **claims** it did.
- **Verify against the original task.** Check for real evidence, not just a "done" sentence:
  - If the task modifies files → read the file / `git diff` (if it's a git repo) and confirm the change is correct and complete per the request.
  - If the task adds a feature/function → confirm the code actually exists, is sensible, and satisfies every requirement in the task message (nothing skipped).
  - If the task runs tests/builds → confirm the result actually appears in the output and passes.
  - If the task is a question/explanation → confirm the answer genuinely answers it and is accurate against the codebase.

### 6. Retry loop if it's not done
If verification shows the task is **incomplete, wrong, or half-finished**, don't immediately report failure to the user — send it back to opencode to redo with `--continue`, specifically explaining what's still missing/wrong:

```bash
opencode run --continue --dangerously-skip-permissions --dir <DIR> "The previous result isn't right yet: <state exactly what's missing/wrong and what's expected>. Please fix it."
```

Then verify again. Repeat this loop until the task is truly done.

Reasonable limit: if after a few rounds (e.g. 2–3) opencode still can't finish, stop and report to the user what's still off, with evidence/output — let the user decide the next step. Don't loop forever.

### 7. Report
Once you're confident it's done (or after giving up within the reasonable limit above), report honestly to the user: what changed, which files, how many rounds it took if relevant, and if anything still failed or was skipped, say so plainly with the relevant output snippet. Don't smooth a failure into a success. Make clear the work was done by opencode and that you verified it.

## Important notes

- **Interactive mode is not for this skill.** Don't run bare `opencode` or `opencode run -i` via bash — the interactive TUI will hang without human input. If the user truly wants an interactive session, suggest they type `! opencode` at the prompt so they drive it themselves.
- **No setup here.** Provider/API key/model are already configured by the user. If opencode errors on credentials/model, report the error — don't try to change opencode's config unless the user asks.
- **Quoting.** Wrap the task message in a single quoted argument. For multi-line messages or ones with tricky shell characters, use single quotes or a heredoc so it isn't mis-parsed.
- **Attribution.** When reporting, be clear that the work was done by opencode (another agent), not by you — so the user knows the source of the changes and can trace them.

## Examples

**Example 1 — direct task**
User: "please add an email validation function in utils, using opencode"
Action:
```bash
opencode run --dangerously-skip-permissions --dir /Users/ilham/atlas "Add an email format validation function in the utils module. Handle common cases (empty, missing @, invalid domain). Follow the existing code conventions in that utils file."
```
Then check the created/changed file and report.

**Example 4 — bug delegation (right vs wrong)**

❌ Wrong — Claude pre-solved it:
```bash
opencode run ... "The auth middleware fails because JWT expiry isn't being checked. Fix it by adding an expiry check in middleware.ts around line 42, comparing token.exp against the current timestamp."
```

✅ Right — opencode solves it:
```bash
opencode run ... "The auth middleware isn't rejecting expired JWT tokens. Fix it."
```

The difference: in the wrong version Claude already diagnosed the cause and prescribed the solution. In the right version the problem is stated and opencode does the investigation and fix.

**Example 2 — continue**
User: "keep going, add unit tests for it"
Action:
```bash
opencode run --continue --dangerously-skip-permissions --dir /Users/ilham/atlas "Add unit tests for the email validation function you just created, covering valid and invalid cases."
```

**Example 3 — codebase question**
User: "have opencode explain the auth flow in this repo"
Action:
```bash
opencode run --dangerously-skip-permissions --dir /Users/ilham/atlas "Explain the end-to-end authentication flow in this codebase: from request entry point, through middleware, to session validation. Name the key files and functions."
```
Summarize the explanation for the user.
