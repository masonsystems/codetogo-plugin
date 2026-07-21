# CodeToGo plugin for Claude Code

Companion slash commands for [CodeToGo](https://codetogo.app), the natural-language front
door to the `codetogo` CLI:

- **`/codetogo:spawn`** — start fresh, titled sessions *now*, one per task, in any project
  directory — driveable from your phone the moment they start.
- **`/codetogo:schedule`** — schedule a fresh, pre-seeded Claude Code session to fire later
  on your own dev machine.
- **`/codetogo:handoff`** · **`/codetogo:resume`** — write a handoff another agent (or a
  fresh session) can pick up, and resume from one.
- **`/codetogo:compact`** — reset a *live* CodeToGo session's context in place: like
  `/compact`, but a real process reset — hot-swap the `claude` process, reseeded from a fresh
  handoff, without a reconnect.

## Prerequisites

The CodeToGo CLI must be installed and running (the plugin shells out to it):

```bash
curl -fsSL https://codetogo.app/install.sh | bash
codetogo login
codetogo start
```

## Install

In Claude Code:

```
/plugin marketplace add masonsystems/codetogo-plugin
/plugin install codetogo@codetogo-marketplace
```

## Spawn

```
/codetogo:spawn fix the flaky auth test
/codetogo:spawn triage the support inbox in ~/src/support-bot, and bump deps in ~/src/site
```

One fresh `claude "<prompt>"` session per task — titled for the task, detached, and live in
your CodeToGo session list immediately, so you can watch, approve, and steer each one from
your phone (assuming `codetogo` is connected to the cloud — the CLI warns at spawn time if
it isn't). Reach for it (instead of in-session team agents/subagents) when you'll drive
the new sessions yourself, or the tasks aren't part of the current session's work — e.g.
one planning session fanning work out across several projects. Same ground rules as
schedule: each prompt is written self-contained (the new session has no memory of the chat
that spawned it), and each directory should be a trusted Claude project.

## Schedule

```
/codetogo:schedule review the open PRs every weekday at 9am
/codetogo:schedule list
/codetogo:schedule remove nightly-review
```

A scheduled run is a **fresh** `claude "<prompt>"` in the chosen directory — it inherits
that dir's files/tools/creds but has **no memory** of the chat that created it, so the
prompt is always written to be self-contained. The directory must be a trusted Claude
project (open Claude there once and accept the trust dialog), and `codetogo` must be
running for the schedule to fire.

## Handoff, resume & compact

Transfer context across a boundary — a new agent, a new session, or a fresh process under
the *same* live session:

```
/codetogo:handoff            # write .claude/tmp/HANDOFF.md (add `quick` for the essentials)
/codetogo:resume [path]      # pick up from a handoff (default: .claude/tmp/HANDOFF.md)
/codetogo:compact [path]       # reset THIS live CodeToGo session in place, reseeded from a handoff
```

`handoff` and `resume` are tool-agnostic — the handoff is a plain `HANDOFF.md` any AI coding
agent can read, kept in the gitignored `.claude/tmp/` and treated as a one-shot baton
(`resume` deletes a default handoff the moment it reads it).

`compact` is CodeToGo-native: inside a CodeToGo-owned session (a `codetogo claude` session, the
web "new session" button, or a scheduled run) it writes a handoff and arms an **in-place
process swap**. At the next idle boundary the old `claude` is killed and
`claude /codetogo:resume <path>` takes its place under the same PTY — same session id,
viewers, phone entry, terminal pane, and scrollback. It's `/compact`, but the reset is a real
new process (a true near-zero context reset) and the client sees no reconnect, only new
output. Reach for it when the context is *polluted*, not merely long.
