# whatsapp

Use Claude to send and read WhatsApp messages — the same way you'd use it
with Gmail or Slack. Cross-platform (macOS/Linux/Windows). No background
agent, no auto-replies: Claude only reads or sends when asked, and sends get
a normal permission prompt first (review before it goes out).

```
"WhatsApp Sam: confirmed for Friday at 3"
"Any new WhatsApp messages from Sam?"
"Catch me up on my WhatsApp"
```

## How it works

The WhatsApp engine is [openclaw/wacli](https://github.com/openclaw/wacli),
a Go CLI built on [whatsmeow](https://github.com/tulir/whatsmeow) / WhatsApp
Web. It pairs as a **linked device** (you link it once with a code from your phone),
keeps a local synced store of your messages, and exposes send/read/search/
group workflows. The plugin ships a thin MCP server over its `--json` CLI, so
WhatsApp works as a plugin in the Claude **desktop** app (not just the
terminal). It runs in every session and exposes:

- **`send_message`** — send to a phone number (`+15551234567`), a WhatsApp
  JID, or a synced contact/group/chat name; text, files with captions,
  quoted replies, and link-preview control (supports attachments ≤100MB)
- **`list_chats`** — an inbox-style view of your most recent conversations
- **`read_messages`** — read conversation history; pass a phone/JID/name for
  a person or group, or omit it for a catch-up across recent threads
- **`search_messages`** — full-text search your synced history ("when did
  Sam mention the invoice?")
- **`react`** — react to a message with an emoji
- **`watch`** — wait briefly for incoming messages and return them (with a
  cursor to resume from), for catching a reply that's expected shortly
- **`list_contacts`** / **`list_groups`** — look up synced contacts and the
  groups you're in (their JIDs are what you address elsewhere)
- **`authenticate`** — link this device to your WhatsApp via a phone pairing
  code, entirely in chat (no QR, no terminal); used by `/whatsapp-setup`
- **`status`** — health check (installed, linked, store reachable)

### The sync wrinkle

wacli reads only what it has **synced** into its local store. The plugin
handles this for you: every read runs a bounded, best-effort `wacli sync`
first, so reads stay fresh without you thinking about it (very recent
messages may lag by a sync pass). For a thread you're actively watching, the
`watch` tool and the `/whatsapp-listen` skill poll across short sync passes.
You can keep the store continuously warm with `wacli sync --follow` in a
terminal; the plugin coexists with it. Disable the auto-sync-before-read with
`WHATSAPP_AUTOSYNC=false` if you prefer to drive sync yourself.

## Setup (one-time, ~5 minutes)

1. The `wacli` engine ships **bundled** with the plugin on macOS (universal
   arm64 + x86_64 binary in `bin/`) — no install step. (Linux/Windows, or as a
   fallback: `brew install openclaw/tap/wacli`, or set `WACLI_PATH`.)
2. **Link your phone, in chat** — tell Claude your WhatsApp number; it gives you
   an 8-character pairing code to enter on your phone (**WhatsApp → Settings →
   Linked Devices → Link a Device → "Link with phone number instead"**). No QR,
   no terminal. wacli then pulls in your recent messages automatically.
3. That's it — no Full Disk Access, no system permission toggles

Run `/whatsapp-setup` (or just say "set up WhatsApp") and Claude walks you
through it conversationally.

| Skill | Does |
| --- | --- |
| `/whatsapp-setup` | One-time setup: verify the bundled `wacli` engine + the in-chat pairing-code link |
| `/whatsapp-send` | Send a message (confirms recipient + exact wording first) |
| `/whatsapp-messages` | Read threads, resolve "Sam" → chat via synced contacts |
| `/whatsapp-listen` | Session-bound watch: "tell me when Alex replies" — polls, notifies, drafts replies for approval, never auto-sends |
| `/whatsapp-status` | Health check |

## ⚠️ Honest caveat — this is your real WhatsApp account

WhatsApp does not officially support third-party Web clients. wacli links as
a normal WhatsApp Web device, but **automating a linked device carries a
low-but-real risk** that WhatsApp flags, rate-limits, or bans the number —
and it's *your* personal/business number, not a throwaway. To keep that risk
low:

- Keep sends **deliberate and human-paced** — no bulk, no tight loops, no
  blasting. wacli prints a warning when sends come too fast for a reason.
- Don't use this for cold outreach, broadcasts, or anything spammy.
- Treat a flag/ban as a possible outcome you've accepted, not a surprise.

This plugin's human-in-the-loop design (every send reviewed, no auto-reply)
is part of keeping usage in the safe zone — but the residual risk is yours.

## Security model

Same consent shape as a mail connector: Claude can read any conversation
**when asked** — linking the device *is* the "Claude may read my WhatsApp"
decision, just like connecting Gmail is the "Claude may read my email"
decision. The guarantees on top:

- Every send is human-approved (permission prompt with the exact text). An
  optional `- Sent by Claude for <name>` AI-disclosure signature is available
  but **off by default** — enable it globally (`signature: true` in
  `~/.claude/whatsapp/config.json`) or per send (`sign_as`). It's off by
  default partly because, on WhatsApp, a visible automation stamp can itself
  raise the risk of the number being flagged.
- Nothing listens for inbound messages; an incoming message cannot trigger
  Claude. Reads happen only when you ask, about what you asked.
- Message content Claude reads is treated as data — instructions embedded in
  messages are surfaced to you, never acted on.
- Reads run in wacli's `--read-only` mode, so they never mutate your account
  or the local store.

## Auto-reply mode

Deliberately **not** part of this plugin's setup. If you want Claude
answering messages unattended, that's an always-on headless process with a
very different risk profile (replies go out unreviewed; the sender allowlist
becomes a real security boundary; and unattended sending sharply raises the
WhatsApp-ban risk above) — see [docs/AUTOREPLY.md](docs/AUTOREPLY.md) for the
recipe and the checklist of caveats.

## Credits

Built on [openclaw/wacli](https://github.com/openclaw/wacli) (MIT) — the
WhatsApp engine that does the pairing, syncing, reading, sending, searching,
and group/contact work. This plugin is the MCP surface and skills around it.

See [NOTICE](NOTICE) for full attribution.

## Support & contributing

Best-effort maintenance — I'll do my best to fix bugs and security issues, and contributions
are very welcome. Open an issue or a pull request; help with docs, tests, or features is
encouraged. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[Apache-2.0](LICENSE) © Ian McDonald.
