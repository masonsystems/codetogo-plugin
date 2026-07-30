---
description: Copy content to the clipboard of the phone or browser viewing this CodeToGo session, as rich text — bold, bullets, and monospace survive a paste into Gmail, Docs, or Slack. Use when the user wants to paste something from this session on their device.
argument-hint: <content to copy> (or omit to copy what was just discussed)
allowed-tools: Bash
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

**Do the whole thing in ONE Bash tool call** — the heredoc below writes the file, pushes it,
and cleans up as several shell statements inside a single call, exactly as shown (it does
*not* need to collapse onto one physical line). Don't use the Write tool for the HTML: that
forces a second turn for no benefit, since the heredoc already preserves the bytes exactly.

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

## 2. Write and push it — one call

```bash
f=$(mktemp /tmp/codetogo-copy-XXXXXX)
cat > "$f" <<'CTG_HTML'
<div style="font-family:Arial,Helvetica,sans-serif;font-size:14px;line-height:1.5">…body…</div>
CTG_HTML
codetogo copy --html "$f" --source "<short label>"; rc=$?; rm -f "$f"; exit $rc
```

Four things in that snippet are load-bearing:

- **The heredoc delimiter is quoted** (`<<'CTG_HTML'`, not `<<CTG_HTML`). Quoting disables
  all shell expansion, so the body's bytes arrive verbatim — apostrophes, em-dashes, smart
  quotes, backticks, `$HOME`, and emoji all survive. An *unquoted* delimiter would expand
  `$…` and backticks and corrupt the content. This is why no escaping or `printf` juggling
  is needed: put the HTML in raw.
- **`mktemp`, not a fixed path.** Two concurrent sessions running this would otherwise
  clobber each other's file.
- **No `.html` suffix on the template.** BSD `mktemp` (the macOS default) only substitutes
  the `X`s when they're at the very END of the template — `…-XXXXXX.html` yields the
  *literal* filename `codetogo-copy-XXXXXX.html`, silently losing the randomization above.
  The CLI doesn't care about the extension, so leave it off.
- **`rm -f` runs after the push, and `rc` preserves the exit code** so a failed push still
  fails the command instead of being masked by the successful `rm`.

The delimiter only ends the heredoc when it's alone on a line — `CTG_HTML` inside a sentence
is ordinary content. In the rare case the body genuinely needs a line that is exactly
`CTG_HTML`, pick a different delimiter.

Write the raw HTML body only — no wrapper tags of your own, no markdown fence.

`--source` is the label on the chip ("Draft email", "Release notes", "psql command") — a
few words, so the user knows what they're about to paste. Omit it if the content is
self-evident.

The CLI derives the plain-text flavor from your HTML automatically and sends **both**, so a
paste into a plain editor still reads correctly. You don't pass the text separately.

## 3. Report

On success the CLI prints how many lines went to how many viewers, e.g.
`Pushed 3 line(s) of rich text to 1 viewer(s)`. Tell the user it's waiting on their device
and that they tap **Copy** on the chip.

If it says **No client is viewing this session**, say so plainly — nothing is watching the
session right now, so the chip has nowhere to land; they should open the session on their
phone or in a browser, then run it again.

### Notes

- Must run **inside** a CodeToGo session — the CLI reads `CODETOGO_SESSION`. Outside one it
  exits with "Not in a CodeToGo session"; pass `--session <id>` to target a specific one.
- **`--html /dev/stdin` also works** (piping the heredoc straight in, no temp file), but
  prefer the `mktemp` form above: if the heredoc is ever omitted, a `/dev/stdin` read can
  block until the turn times out, and the temp file has no such failure mode.
- If the success line says `N line(s) of text` rather than `of rich text`, the running
  daemon predates the rich flavor and dropped it — tell the user to run `codetogo restart`
  and try again.
- 128 KB limit across both flavors. For anything larger, point the user at the file instead
  of pasting it.
- Plain text is fine too: `codetogo copy "some text"` or `codetogo copy --file <path>` skips
  the HTML entirely. Reach for `--html` when formatting is the point.
- The chip stays until the user copies or dismisses it, so this works even if their phone is
  in their pocket right now.
