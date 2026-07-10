---
description: Reset THIS live CodeToGo session's context in place — like /compact, but a real process reset reseeded from a handoff
argument-hint: "[handoff-path]"
---

CodeToGo's counterpart to `/compact`. Where `/compact` summarizes the context *in place*
(same polluted process, now with a summary on top), this does a **hard reset**: it replaces
the `claude` process running under **this** CodeToGo session with a fresh one — same terminal
pane, same session id, same viewers, same phone entry, same scrollback — reseeded from a
handoff you write now. The "summary" is a curated handoff, and the reset is a **real new
process** (context truly reset to near-zero). Reach for it when the context is *polluted*,
not merely long.

The swap fires at the next idle boundary — i.e. **when this turn ends** — via CodeToGo's
Stop hook: this turn writes the handoff and arms the compact; the moment you finish, the old
`claude` is killed and `claude /codetogo:resume <path>` takes its place under the same PTY.
The client sees no reconnect, only new output.

## HARD RULE — this command is a stop order

The whole point of this command is to stop spending tokens in this process **immediately**.
From the moment it's invoked, your only remaining job is: write the handoff, arm the compact,
end the turn. That is exactly **three tool calls** — the Bash call in step 1, the Write of the
handoff in step 2, the Bash call in step 3 — and **nothing else**.

Do **not** use any other tool or do any other work: don't finish the in-flight task, don't
run tests, don't gather state (`git status`, `git diff`, log searches, file reads), don't
tidy up, don't investigate. Anything you were doing — or wanted to do — becomes a **Not Yet
Done** item in the handoff for the next agent; the fresh process does it, not you. Every
extra tool call here adds a turn and spends the very tokens this command exists to save.

## 1. Precondition + handoff path — ONE Bash call

```bash
if [ -z "$CODETOGO_SESSION" ]; then echo "NOT a codetogo-owned session"; else ROOT="${CLAUDE_PROJECT_DIR:-}"; [ -z "$ROOT" ] && ROOT="$(git rev-parse --show-toplevel 2>/dev/null)"; if [ -z "$ROOT" ]; then d="$PWD"; while [ "$d" != "/" ] && [ ! -d "$d/.claude" ]; do d=$(dirname "$d"); done; ROOT="$d"; fi; [ "$ROOT" = "/" ] && ROOT="$PWD"; mkdir -p "$ROOT/.claude/tmp"; echo "owned: $CODETOGO_SESSION"; echo "transcript: cc/$CLAUDE_CODE_SESSION_ID"; echo "handoff: $ROOT/.claude/tmp/HANDOFF.md"; fi
```

This only works when CodeToGo spawned the PTY — a `codetogo claude` session, the web "new
session" button, or a scheduled session. A bare `claude` that CodeToGo only sees via hooks
is **not** swappable. If it prints `NOT a codetogo-owned session`, stop and tell the user
this command only works inside a CodeToGo-managed session — there's nothing to reset.

## 2. Write the handoff — from context only

Write a complete `# Handoff: <title>` document (the format `/codetogo:handoff` defines) to
the `handoff:` path from step 1, built **entirely from what's already in your context**. Run
nothing to gather state — the next agent can run `git status` itself for one cheap call; you
re-deriving it now defeats the purpose. If you don't know something (exact diff state,
whether tests pass), *say so in the handoff* rather than checking. Include the `transcript:`
id from step 1 so the next agent can consult the source conversation.

Everything unfinished — including whatever this command interrupted — goes under **Not Yet
Done** / **Resume Instructions** as standing orders for the next agent.

If `$ARGUMENTS` is a path, the user already wrote/reviewed the handoff — skip this step and
use that path instead.

## 3. Arm the compact, then end the turn

Arm the in-place swap for THIS session, pointing at the handoff's **absolute** path — the
explicit path guarantees the fresh `claude` loads *this* handoff regardless of the dir it
respawns in (a no-arg resume resolves the default path from cwd, which lands in the wrong tree
for a worktree session — COD-811). The resumed session deletes the baton after reading it:

```bash
codetogo compact "<absolute-handoff-path>"
```

Expect `Session compact armed.` Then **end your turn immediately** — one line to the user
("Handoff written, compact armed — resetting now."), no work summary, no follow-ups. The
swap fires on the next Stop hook, so an in-flight turn is never killed. If it instead prints
`Not in a CodeToGo session` or `Failed to arm session compact: …`, the step-1 precondition
wasn't actually met (or the server isn't running) — report exactly what it said and stop.
