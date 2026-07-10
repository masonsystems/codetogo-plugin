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

## 2. Delete the handoff if it's the one-shot baton — before any work

A handoff at `.claude/tmp/HANDOFF.md` is a **one-shot baton**: `/codetogo:handoff` writes there,
and a `/codetogo:compact` swap re-seeds this session by pointing `/codetogo:resume` at that exact
path. So the instant you've read it, delete it — right now, before summarizing, before touching
code — **whether you loaded it by default (no arg) or via an explicit `$ARGUMENTS` path**. Its
content is already in your context; a lingering `.claude/tmp/HANDOFF.md` is the #1 cause of
"which handoff is actually resuming?" confusion and gets silently re-loaded (and deleted) by the
next no-arg resume. Deleting it is safe: the compact swap fires exactly once, so there is no
re-fire that still needs the file.

The discriminator is the file's location, **not** how the path was passed. Delete the exact file
you loaded only when its path is that ephemeral baton (ends in `.claude/tmp/HANDOFF.md`):

```bash
HP="$ARGUMENTS"; if [ -z "$HP" ]; then ROOT="${CLAUDE_PROJECT_DIR:-}"; [ -z "$ROOT" ] && ROOT="$(git rev-parse --show-toplevel 2>/dev/null)"; if [ -z "$ROOT" ]; then d="$PWD"; while [ "$d" != "/" ] && [ ! -d "$d/.claude" ]; do d=$(dirname "$d"); done; ROOT="$d"; fi; [ "$ROOT" = "/" ] && ROOT="$PWD"; HP="$ROOT/.claude/tmp/HANDOFF.md"; fi; case "$HP" in .claude/tmp/HANDOFF.md|*/.claude/tmp/HANDOFF.md) rm -f "$HP" && echo "deleted one-shot handoff: $HP" ;; *) echo "kept persistent doc (not a .claude/tmp baton): $HP" ;; esac
```

Exception: an explicit `$ARGUMENTS` path that points **elsewhere** — anything not ending in
`.claude/tmp/HANDOFF.md` (a doc at the repo root, under `docs/`, a saved plan) — is a persistent
doc the user pointed at deliberately; the `case` above leaves it untouched.

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
