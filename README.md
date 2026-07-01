# CodeToGo plugin for Claude Code

The `/codetogo:schedule` slash command — a natural-language front door to
[`codetogo schedule`](https://codetogo.app). Describe **what** to do and **when**, and
it captures the current directory, writes a self-contained prompt, previews it with
`--dry-run`, and (on confirmation) schedules a fresh Claude Code session to fire later
on your own dev machine.

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

## Use

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
