---
description: Copy content to the clipboard of the phone or browser viewing this CodeToGo session, as rich text — bold, bullets, and monospace survive a paste into Gmail, Docs, or Slack. Use when the user wants to paste something from this session on their device.
argument-hint: <content to copy> (or omit to copy what was just discussed)
allowed-tools: Bash, Write
---

Put `$ARGUMENTS` on the clipboard of whatever device is **viewing this session** — the
user's phone, or a browser tab at https://codetogo.app — formatted as rich text so a paste
into Gmail/Docs/Slack keeps the formatting instead of arriving as a wall of asterisks.

This is the remote twin of a local "copy to clipboard": the agent runs on the dev machine,
but the clipboard that gets written is the one on the device the user is holding. The user
taps **Copy** on the chip that appears over their terminal; that tap is what writes the
clipboard (browsers only allow a clipboard write inside a real user gesture).

If `$ARGUMENTS` is empty, copy the thing just produced or discussed — a draft email, a
summary, a table, a command. Ask only if that's genuinely ambiguous.

## 1. Convert the content to HTML

- `**text**` / `__text__` → `<b>text</b>`
- `*text*` / `_text_` → `<i>text</i>`
- Newlines → `<br>`
- Bullet lines starting with `-` or `*` → `• ` (a real bullet char, U+2022)
- URLs stay plain text — mail clients auto-link them
- Inline code and fenced code blocks → `<span style="font-family:'Courier New',monospace">…</span>`,
  **not** `<code>` (Gmail strips it)

**Wrap the whole body in one div** so a paste looks like a normally composed message rather
than the browser's default Times:

```html
<div style="font-family:Arial,Helvetica,sans-serif;font-size:14px;line-height:1.5">…body…</div>
```

Arial 14px is Gmail's "Sans Serif / Normal". Code spans inherit the size, so keep them
monospace per the rule above.

## 2. Write it to a temp file

Use the Write tool with a **randomized, never-before-used filename** —
`/tmp/codetogo-copy-<random>.html` (append a few random digits or a timestamp). Do NOT
reuse a fixed name: the Write tool refuses to overwrite a file it hasn't read this session,
so the second run of a fixed name fails and costs a retry. Write the raw HTML body only —
no wrapper tags of your own, no markdown fence.

Going through a file (not a shell variable) is what makes apostrophes, em-dashes, quotes,
and non-ASCII survive — shell quoting mangles all of them.

## 3. Push it to the device

```bash
codetogo copy --html /tmp/codetogo-copy-<random>.html --source "<short label>"
```

`--source` is the label on the chip ("Draft email", "Release notes", "psql command") — a
few words, so the user knows what they're about to paste. Omit it if the content is
self-evident.

The CLI derives the plain-text flavor from your HTML automatically and sends **both**, so a
paste into a plain editor still reads correctly. You don't pass the text separately.

Then remove the temp file:

```bash
rm -f /tmp/codetogo-copy-<random>.html
```

## 4. Report

On success the CLI prints how many lines went to how many viewers. Tell the user it's
waiting on their device and that they tap **Copy** on the chip. If it says **0 viewers**,
say so plainly — nothing is watching the session right now, so the chip has nowhere to
land; they should open the session on their phone or in a browser and run the command
again.

### Notes

- Must run **inside** a CodeToGo session — the CLI reads `CODETOGO_SESSION`. Outside one it
  exits with "Not in a CodeToGo session"; pass `--session <id>` to target a specific one.
- **Older CLI:** `--html` needs a recent `codetogo`. If the command fails with
  `unknown option '--html'`, write the *plain-text* version of the same content to a second
  temp file and push that with `codetogo copy --file <plain.txt>` — never `--file` the HTML
  file itself, which would paste raw markup. Tell the user it went as plain text and that
  `codetogo upgrade` enables the formatted version.
- 128 KB limit across both flavors. For anything larger, point the user at the file instead
  of pasting it.
- Plain text is fine too: `codetogo copy "some text"` or `codetogo copy --file <path>` skips
  the HTML entirely. Reach for `--html` when formatting is the point.
- The chip stays until the user copies or dismisses it, so this works even if their phone is
  in their pocket right now.
