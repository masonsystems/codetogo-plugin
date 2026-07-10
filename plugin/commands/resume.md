---
description: Resume work from a handoff written by /codetogo:handoff (also what an in-place context reset re-seeds into)
argument-hint: "[path]"
---

Pick up work from a handoff. This is also the command a `/codetogo:compact` swap re-seeds the
fresh `claude` process with, so keep it robust.

## 1. Load the handoff

Resolve in priority order:

1. **Explicit path** — if `$ARGUMENTS` is set, read that path. Trusted; load it directly.
2. **Default** — resolve the project root (`$CLAUDE_PROJECT_DIR`, else `git rev-parse
   --show-toplevel`, else walk up from the cwd to the nearest `.claude/`) and read
   `.claude/tmp/HANDOFF.md` there. A real handoff's first line is `# Handoff:`.
3. Nothing found → ask the user for the path. Don't hunt for stray `HANDOFF.md` files — one
   at the repo root or cwd is a persistent doc, not a session handoff; load it only if the
   user points you at it explicitly.

Handoffs are **not** auto-loaded — a new session picks one up only when you run
`/codetogo:resume` (optionally with a path). Read the whole document.

## 2. Delete a default handoff immediately — before any work

The instant you've read a **default** handoff (the `.claude/tmp/HANDOFF.md` loaded by
default), delete it — right now, before summarizing, before touching
code. It's a one-shot baton: its content is already in your context, and a lingering file is
the #1 cause of "which handoff is actually resuming?" confusion.

```bash
ROOT="${CLAUDE_PROJECT_DIR:-}"; [ -z "$ROOT" ] && ROOT="$(git rev-parse --show-toplevel 2>/dev/null)"; if [ -z "$ROOT" ]; then d="$PWD"; while [ "$d" != "/" ] && [ ! -d "$d/.claude" ]; do d=$(dirname "$d"); done; ROOT="$d"; fi; [ "$ROOT" = "/" ] && ROOT="$PWD"; rm -f "$ROOT/.claude/tmp/HANDOFF.md"
```

Exception: if `$ARGUMENTS` gave an explicit path, leave that file in place — the user (or a
compact swap that may need to re-fire) pointed at it deliberately.

## 3. Get to work

A compact swap re-execs a fresh process under the *same* PTY and checkout with ~zero elapsed
time, so there's nothing to reconcile against the repo — go straight to the work. Give a
one-line summary and **start on the outstanding work immediately** — **Not Yet Done** /
**Resume Instructions** are standing orders the user already gave. Don't end on "Ready to
continue?" and wait.

```
Resuming from handoff: <title>
Goal: <1 sentence> · Status: <X of Y done>
Picking this up now — starting with <first item>.
```

Pause for the user only when: a blocker needs a
decision/credentials/access only they can give, the outstanding work is genuinely ambiguous
**and** the source transcript doesn't resolve it (`aii show cc/<id>`), or there's no
actionable work left (pure context) — then say it's loaded and ask what they want.

Heed **Failed Approaches** (don't repeat them), **Warnings**, and **Key Decisions**.
