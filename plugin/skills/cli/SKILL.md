---
name: cli
description: "BLOCKING: read this BEFORE running any `codetogo` command or writing any jq over its output — the field names are NOT guessable and a wrong guess returns empty, which reads as \"no such session\" and is a false negative. Triggers: any mention of a CodeToGo/PTY/agent session on this machine, a session or conversation or transcript id, \"which session/agent did this\", \"what is that session doing\", \"is anything waiting on me\", listing/finding/reading sessions, or grepping ~/.claude/projects by hand. Covers: mapping a conversation id to its session (and back), reading a session without attaching, triaging what needs the user."
---

# CodeToGo CLI

`codetogo` runs a local server on this machine that owns every CodeToGo session (each
one a PTY, usually running `claude` or `codex`) and relays it to the user's phone and
browser. The CLI is the read/write surface for that server.

**Start here for any session question:** `codetogo sessions --json` — one call, machine
readable, and it carries the id mapping that makes everything else possible.

```bash
codetogo sessions --json
```

```json
[
  {
    "id": "b6899618-ef6a-4813-9936-e25f46e6c3db",
    "shortId": "b6899618",
    "state": "working",
    "rawState": "tool_execution",
    "displayName": "Qa brenda",
    "cwd": "/Users/eric/src/hourglass",
    "agentSessionId": "f077f7c8-f298-4792-b0ae-35c6dfb2c9ec",
    "stateSinceIso": "2026-07-31T14:22:46.993Z",
    "createdAt": "2026-07-30T19:18:01.109Z",
    "hostConnected": true,
    "clientCount": 0,
    "needsAttention": false,
    "type": "terminal"
  }
]
```

Prefer `--json` over the human listing in every automated read: the plain output is
`[codetogo] `-prefixed and reflowed, so parsing it is strictly worse.

**Those are all the field names there are.** Do not guess one. `jq` on a field that does
not exist prints nothing and exits 0, which looks exactly like "no session matches" — so a
guessed key does not fail loudly, it fabricates a false negative. The plausible-sounding
names that do **not** exist: `conversationId`, `claudeSessionId`, `sessionId`, `agentId`,
`name` (it's `displayName`), `title`, `dir`, `status` (it's `state`). If a lookup comes back
empty, re-read the field list above before concluding the session isn't there.

## The two ids — read this before anything else

Every session has **two** unrelated uuids, and mixing them up is the single most common
way to waste turns here:

| Field | What it is | Where it's valid |
|---|---|---|
| `id` / `shortId` | The **PTY / CodeToGo session** id | `codetogo tail`, `viewers`, `connect`, `copy --session`, `logs --session`, the `?session=` deep link |
| `agentSessionId` | The **agent conversation** id (Claude Code / Codex) — names the transcript on disk | `~/.claude/projects/*/<id>.jsonl`, `claude --resume <id>`, `~/.codex/sessions/**/*-<id>.json` |

They are **not** interchangeable and neither command falls back to the other:

```bash
codetogo tail f077f7c8-f298-4792-b0ae-35c6dfb2c9ec   # ✗ "No session matching ..."
codetogo tail b6899618                               # ✓ (PTY id — prefixes are fine)
codetogo logs search --session <PTY-id>               # ✓ needs the FULL PTY uuid, not a prefix
```

`agentSessionId` is the **current** binding, not history. A session that was restarted or
`/codetogo:compact`ed has a new agent id; its older ids are only in the resume snapshot
(see "Older / dead sessions" below).

## Recipes

Every `jq` below assumes the bare-array shape above.

**Which session produced this work?** (transcript id → the live session, its title and dir).
This is the reverse lookup that makes "which agent wrote that commit" a one-liner instead
of a transcript grep:

```bash
AG=f077f7c8-f298-4792-b0ae-35c6dfb2c9ec
codetogo sessions --json | jq -r --arg a "$AG" \
  '.[] | select(.agentSessionId==$a) | "\(.shortId)  \(.displayName)  \(.cwd)"'
```

**Is anything waiting on the user?** — the triage question:

```bash
codetogo sessions --json | jq -r \
  '.[] | select(.state=="waiting-on-you" or .state=="blocked-on-you" or .state=="error")
   | "\(.shortId) \(.state) \(.displayName) — \(.cwd)"'
```

**What's running in a given project?**

```bash
codetogo sessions --json | jq -r --arg d "$PWD" \
  '.[] | select(.cwd==$d) | "\(.shortId) \(.state) \(.displayName) agent=\(.agentSessionId)"'
```

**What did a session just say?** — reads the transcript off disk; does **not** attach,
steal the PTY, or disturb the session:

```bash
codetogo tail <pty-id-prefix>            # last 3 assistant messages
codetogo tail <pty-id> -n 10 --max-lines 0   # more turns, untruncated
```

Prints the title, cwd, and `claude|codex <agentSessionId>` header, so it doubles as a
PTY-id → agent-id lookup for one session. Works for Claude and Codex sessions alike.

**Transcript path from an agent id** (for reading raw JSONL yourself). Claude Code
replaces every char outside `[a-zA-Z0-9-]` in the cwd with `-`:

```bash
# ~/.claude/projects/<cwd with non-alnum → ->/<agentSessionId>.jsonl
ls ~/.claude/projects/-Users-eric-src-hourglass/$AG.jsonl
# Codex instead: ~/.codex/sessions/**/rollout-<date>-<agentSessionId>.json
```

**Open a session on the user's phone/browser** — the `serverId` for the link lives in
`~/.codetogo/auth.json`:

```bash
SRV=$(jq -r .serverId ~/.codetogo/auth.json)
codetogo sessions --json | jq -r --arg s "$SRV" \
  '.[] | "\(.displayName): https://codetogo.app/terminal?server=\($s)&session=\(.id)"'
```

**Other reads:** `codetogo status` (login, server pid/port, cloud latency, all sessions
with deep links) · `codetogo viewers [pty-id]` (who's watching + the PTY size, and the
per-viewer sizes that explain a clamped width) · `codetogo logs search "<text>" --since 1h`
(central logs across CLI, cloud, and relay — the user's phone has no console, so this is
the only way to see client-side logs).

## Reading `state`

`state` is a derived, coarse verdict — use it, not `rawState`, for decisions:

- `working` — actively thinking/streaming/running a tool.
- `waiting-on-you` — a permission prompt or a fired attention signal. **Act on this.**
- `blocked-on-you` — finished its turn on an unanswered question. Silent otherwise, so
  this is the one that hides sessions stuck since Friday.
- `idle` — at a prompt, nothing pending.
- `error` / `dead` — errored, or the host side is gone.
- `unknown` — **no detector for this session**, not "fine". Its state was wiped (e.g. by
  a server restart) and nothing has re-established it, so it may well be parked on
  something. Don't report `unknown` as idle.

`stateSinceIso` is the last state *change* (null = never transitioned), not the time of
your call — so "working for 4 hours" is a real, computable signal. It is **UTC**; convert
before showing the user a time.

## Acting on the session you're running inside

An agent running inside a CodeToGo session has `CODETOGO_SESSION` set to its own **PTY
id**, and these commands target that session implicitly — no id argument:

```bash
codetogo rename "Fix flaky auth test"    # retitle this session in the user's list
codetogo snooze [until-wake|1h|4h|8h|clear]  # stop surfacing this session as waiting
codetogo copy --md notes.md              # rich text → the clipboard of the device viewing this session
codetogo copy --md notes.md --session <pty-id>   # ...or aim it at another session
```

`snooze` is for when you've armed a wait CodeToGo can't see — a cron, a CI run, a promised
follow-up — so the session doesn't sit in the user's list looking like it needs them.

Outside a CodeToGo session `CODETOGO_SESSION` is unset and these exit 1 with "Not in a
CodeToGo session" — that's the honest answer, not a bug to work around. `copy` is the one
exception: `--session <pty-id>` overrides the env var, so it works from anywhere.

**`codetogo notify "deploy finished"` is NOT one of these.** It pushes to the user's
devices account-wide, reads no session id at all, and works fine outside a session — so
don't reach for `CODETOGO_SESSION` before calling it.

Related, and covered by their own commands rather than raw CLI: **`/codetogo:spawn`**
(start new titled sessions, one per task), **`/codetogo:schedule`** (fire a pre-seeded
session later), **`/codetogo:copy`**, **`/codetogo:handoff`** · **`/codetogo:resume`** ·
**`/codetogo:compact`**. Prefer those over hand-rolling `codetogo spawn` / `schedule add`.

## Older / dead sessions

`sessions --json` lists only what's **live now**. For a session that has since ended,
read the snapshots the server writes (`~/.codetogo/snapshots/`, also `codetogo resume
--list`):

```bash
codetogo resume --json | jq -r '.sessions[]
  | select(.agentSessionIds[]? | startswith("019fb34f"))
  | "\(.name)  \(.cwd)  \(.agentType)"'
```

Unlike the live row, snapshot entries keep **`agentSessionIds` as a list** — every agent
conversation that PTY ever hosted — so this is the only place a pre-restart or
pre-compact agent id can still be resolved to a project.

## Hazards

- **Don't hand an `agentSessionId` to a PTY-id command** (or the reverse). Both fail as
  "not found", which reads as "that session doesn't exist" and sends you off grepping.
- **Prefixes:** `tail` / `viewers` / `connect` accept an unambiguous PTY-id prefix and
  name the collisions when there are several. `logs search --session` does **not** —
  it silently returns "No logs found" for a prefix. Pass the full uuid.
- **Read-only vs. intrusive:** `sessions`, `tail`, `viewers`, `status`, `logs` are safe on
  a session someone is using. `codetogo connect` **attaches your terminal to a live PTY**
  and, with no argument, auto-selects when there's exactly one session — never reach for
  it to "just look"; `tail` is the read.
- **Never run `codetogo stop`** to fix something. It kills every PTY on the machine and
  cuts the user off from their phone. `codetogo restart` preserves sessions.
- **`codetogo logs` reads *production* by default** even while you're testing elsewhere;
  each line carries a `prod`/`stg` badge. Tag the environment before drawing conclusions.
- **`sessions --json` exits 1 with `{"error":"server-not-running"}`** when the server is
  down — check the exit code, and don't misread the empty listing as "no sessions".
- **All timestamps are UTC.** Convert to the user's local zone before showing them.
- **`--json` needs CLI ≥ 1.2.32** (`codetogo -V`). On an older CLI the flag is rejected;
  `codetogo upgrade` is the fix.
