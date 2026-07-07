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

## 0. Precondition — must be a CodeToGo-owned session

```bash
[ -n "$CODETOGO_SESSION" ] && echo "owned: $CODETOGO_SESSION" || echo "NOT a codetogo-owned session"
```

This only works when CodeToGo spawned the PTY — a `codetogo claude` session, the web "new
session" button, or a scheduled session. A bare `claude` that CodeToGo only sees via hooks
is **not** swappable. If it prints `NOT a codetogo-owned session`, stop and tell the user
this command only works inside a CodeToGo-managed session — there's nothing to reset.

## 1. Write the handoff

Do exactly what `/codetogo:handoff` does — gather state (`git status`, `git diff --stat`,
`git log --oneline -5`, `echo "cc/$CLAUDE_CODE_SESSION_ID"`), mine the conversation, and
write a complete `# Handoff: …` document. Save it under the project root and capture the
absolute path:

```bash
ROOT="${CLAUDE_PROJECT_DIR:-}"; [ -z "$ROOT" ] && ROOT="$(git rev-parse --show-toplevel 2>/dev/null)"; if [ -z "$ROOT" ]; then d="$PWD"; while [ "$d" != "/" ] && [ ! -d "$d/.claude" ]; do d=$(dirname "$d"); done; ROOT="$d"; fi; [ "$ROOT" = "/" ] && ROOT="$PWD"; mkdir -p "$ROOT/.claude/tmp"; echo "$ROOT/.claude/tmp/HANDOFF.md"
```

Write to the absolute path that prints; the first line must be `# Handoff: <title>`. If
`$ARGUMENTS` is a path, the user already wrote/reviewed the handoff — skip writing and use
that path instead.

## 2. Arm the compact

Arm the in-place swap for THIS session, pointing at the handoff's **absolute** path (use the
explicit-path form — the no-arg resume deletes the handoff on read, which would leave nothing
if the swap needs to re-fire):

```bash
codetogo compact "<absolute-handoff-path>"
```

Expect `Session compact armed.` Then **end your turn** — the swap fires on the next Stop
hook, so an in-flight turn is never killed. If it instead prints `Not in a CodeToGo session`
or `Failed to arm session compact: …`, the step-0 precondition wasn't actually met (or the
server isn't running) — report exactly what it said and stop.
