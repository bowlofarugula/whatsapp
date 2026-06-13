---
name: whatsapp-send
description: Send a WhatsApp message to someone from this session. Use when the user asks to WhatsApp someone, send a WhatsApp message, message a person on WhatsApp, or "tell X" something over WhatsApp. Claude confirms the exact text before sending.
user-invocable: true
---

# /whatsapp-send — send a WhatsApp message

Arguments passed: `$ARGUMENTS` (free-form, e.g. "Sam let's confirm Friday"
or "+15551234567 the cut is ready")

## Resolve the recipient

You need an address: a phone number in `+country` format
(`+15551234567`), a WhatsApp JID (`…@s.whatsapp.net` for a person,
`…@g.us` for a group), or a **synced contact/group/chat name**. wacli
resolves names and phone numbers itself, so you can usually pass what the
user said straight through as `to`.

- A name you're unsure about → confirm with `list_contacts` (search) or
  `list_chats` first, so you address the right person. If a name is
  ambiguous, wacli returns multiple matches — pass `pick: N` to choose one.
- A brand-new number with no prior chat → a phone number in `+` format
  works directly.

## Confirm, then send

**Always show the exact recipient and exact message text and get a yes
before sending.** If the user dictated rough intent ("tell her I'm running
late"), draft the text, show it, let them edit.

Send with the **`send_message` MCP tool** (this plugin's WhatsApp server):

- `to` — phone / JID / synced name (above).
- `text` — the message, sent as written (no signature by default; see below).
- `files` — absolute paths to attach (≤100MB each); optional `caption` on
  the first file.
- `reply_to` — quote a message id (the `{id:…}` shown by `read_messages`).
- `link_preview` — default true; set false to suppress the URL preview.

If the **approval gate** is enabled (`approval: true` in
`~/.claude/whatsapp/config.json`), the send also pauses on a client dialog
showing the recipient and final text — only the user can answer it, and it
is the final say. A "cancelled" tool result means they declined: stop
there, don't retry or rephrase to resend. If the result says the client
can't show approval prompts, relay that message as-is — the send was
blocked on purpose (fail-closed), not broken.

## Signature (optional, OFF by default)

Messages go out **as-is** unless the user opted into the
`- Sent by Claude for <name>` disclosure. Don't add disclosure text yourself
and don't nag about it. On WhatsApp there's an extra reason to leave it off: a
visible "this is automated" stamp on every message is the kind of signal that
gets numbers flagged — so it's off by default deliberately.

- It's on for a send when the user enabled it globally (config `signature: true`
  or env `WHATSAPP_APPEND_SIGNATURE=true`), or when you pass `sign_as` for that
  send. When it's on, include the signature line in the confirmation you show.
- "Send this as Acme" / "sign it from the studio" → pass `sign_as: "Acme"` on
  that send only (this also turns the signature on for just that message).
- "Always sign my messages" / "turn the signature on" → set `signature: true`
  (and optionally `signatureName`) in `~/.claude/whatsapp/config.json` (create
  the file/dir if absent; read first and preserve other fields; 2-space
  indent). Applies from the next send — the server re-reads per call.
- "Stop signing" / "turn it off" → set `signature: false` (or remove it).
- When on, the name is: `sign_as` > config `signatureName` > OS account name.
- Don't drop "Sent by Claude" from a signature once one is being used —
  only the name varies.

## Groups

Reach a group by its JID (`…@g.us`) or its synced name. Find it with
`list_groups` or `list_chats` (group rows show the name and JID); pass that as
`to`. Claude can message any group the linked account is already in. Creating
or leaving groups isn't part of this plugin's send flow — do that in WhatsApp.

## Pace and account safety

This drives the user's **real** WhatsApp number as a linked device. Keep
sends deliberate and human-paced — one considered message at a time, no bulk
or rapid-fire loops. If a send warns about rate/too-fast, stop and slow down;
the ban risk on a linked device is real.

## wacli CLI fallback

Only if the MCP tools aren't available in this session (e.g. setup
incomplete). The plugin's launcher at `${CLAUDE_PLUGIN_ROOT}/bin/wacli`
finds the installed engine (`~/.claude/whatsapp/engine`, or a brew `wacli`
on PATH):

```sh
"${CLAUDE_PLUGIN_ROOT:-.}/bin/wacli" send text --to "<phone-or-jid-or-name>" --message "<text>"
```

Send the message as the user wrote it — no signature on this path either,
unless they asked for one (then append two newlines + `- Sent by Claude for
<name>`). Attach a file with
`… send file --to "<recipient>" --file "<path>" --caption "<text>"`.

## Failure modes

- error mentions `not linked` / `pair` → the device session lapsed (often
  unlinked from the phone). Run `/whatsapp-setup` (or the `authenticate` tool)
  to re-link with a pairing code.
- error mentions the store `lock` → another wacli process (often
  `sync --follow`) holds it; stop it or wait, then retry.
- error mentions `rate` / `too many` / `forbidden` → WhatsApp is throttling
  or flagging the number; stop sending and let it cool off.
