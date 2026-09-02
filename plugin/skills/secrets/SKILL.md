---
name: secrets
description: "Ask the user for an API key, token, password, or any other credential you need to finish a task — `codetogo secret request`. Use this INSTEAD of telling the user to paste a key into the chat, put one in a file, or edit .env themselves: they are usually on a phone and cannot edit files, and a key pasted into the conversation is recorded permanently in the transcript, the terminal scrollback, and the logs. Triggers: you encounter a missing ANTHROPIC_API_KEY / OPENAI_API_KEY / GITHUB_TOKEN / AWS credential / database password / .env value, a command fails with 401 or 403 or \"not authenticated\", a setup step needs a credential you do not have, or you are about to write \"please add your key to\" anything."
---

# Ask the user for a secret

When you need a credential to continue, ask for it directly. The user gets a masked field on whatever device they are holding, pastes the value, and you get back a file path.

```bash
codetogo secret request ANTHROPIC_API_KEY --reason "Running the deploy script"
```

The command blocks until the user answers, then prints the path on its own line:

```
Got ANTHROPIC_API_KEY. Read it, use it, then delete the file:
/Users/eric/.codetogo/uploads/<session>/secrets/<id>-ANTHROPIC_API_KEY
```

Read the file, put the value where it belongs, and delete the file:

```bash
SECRET_PATH="$(codetogo secret request ANTHROPIC_API_KEY --reason "Deploy needs it" | tail -1)"
printf 'ANTHROPIC_API_KEY=%s\n' "$(cat "$SECRET_PATH")" >> .env
rm -f "$SECRET_PATH"
```

## Rules

- **Never print the value.** No `cat` of the file to the terminal, no echoing it back to confirm, no putting it in a commit message or a log line. Everything you print is recorded where the value must not be.
- **Use shell substitution, never your own eyes.** `$(cat "$SECRET_PATH")` moves the value without it entering the conversation. Reading the file yourself puts the credential in your context, which is exactly what this avoids.
- **Delete the file when you are done.** It self-deletes after five minutes, but that is a backstop, not a plan.
- **Say what it is for.** `--reason` is shown above the field. A request to paste a credential with no stated purpose is one the user should refuse, so give them what they need to say yes.
- **Name the variable, not a description.** `ANTHROPIC_API_KEY`, not `the API key`. The name is shown to the user and becomes the file name. Letters, digits and underscores only.

## When the user says no

Every ending is explicit and exits non-zero, so you can branch on it:

| Ending | What it means | What to do |
|--------|---------------|------------|
| `Declined` | The user chose not to provide it | Stop asking. Say what you cannot do without it. |
| `Timed out` | Nobody answered within the window (default 10 minutes) | Say you are blocked and what you need; do not loop. |
| `The session closed` | The session went away while waiting | Nothing to do. |

Do not retry a decline. The user answered.

## Options

| Flag | Purpose |
|------|---------|
| `--reason <text>` | One line on what the secret is for, shown above the field |
| `--timeout <seconds>` | How long to wait (default 600) |
| `--session <id>` | Target another session; defaults to the one you are running in |

## When not to use this

- **The value is already available.** Check the environment and the existing config first — asking for a key the machine already has wastes the user's attention.
- **You do not actually need it.** If the task can be finished without the credential, finish it.
- **It is not a secret.** A path, a URL, a project name, or any other non-sensitive answer is an ordinary question. Ask it in the conversation.
