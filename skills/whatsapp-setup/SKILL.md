---
name: whatsapp-setup
description: One-time setup for WhatsApp from Claude — verifies the bundled wacli engine and links the device with an in-chat pairing code (no QR, no terminal) via the authenticate tool, then the initial sync. Use when the user asks to set up WhatsApp, when the WhatsApp tools (send_message/read_messages) are missing or erroring, or before the first send/read on a new machine.
user-invocable: true
---

# /whatsapp-setup — WhatsApp connector setup

After this runs, the user can say "WhatsApp Sam we're confirmed for Friday" or
"any new WhatsApp messages from Sam?" in any Claude session, with normal
permission prompts before anything is sent. There is no background agent and no
auto-reply: Claude only reads or sends when asked.

The WhatsApp engine is [`wacli`](https://github.com/openclaw/wacli), a Go CLI
that pairs as a linked WhatsApp Web device; the plugin ships a thin MCP server
over it that exposes `authenticate` (device linking), `send_message`,
`list_chats`, `read_messages`, `search_messages`, `react`, `watch`, and
`status` in every session.

The user may be non-technical and is probably in the **Claude desktop app, not
a terminal**. Linking happens **right here in chat** — no terminal, no QR to
scan. Run every check yourself; the only thing they do by hand is enter one
short code on their phone.

Arguments passed: `$ARGUMENTS` (optional)

---

## Step 0 — say the honest caveat first

Before installing anything, tell the user plainly, once:

> This links Claude to your **real** WhatsApp account as a Web device.
> WhatsApp doesn't officially support third-party Web clients, so automating
> a linked device carries a low-but-real risk that your number gets
> rate-limited or banned. Keep messages deliberate and human-paced — no bulk
> or spammy sends — and that risk stays low. OK to continue?

Only proceed once they're OK with it.

## Step 1 — the wacli engine

**macOS:** `wacli` ships **bundled inside this plugin** (`bin/darwin/wacli`, a
universal arm64 + x86_64 binary), so there's normally no install step. The MCP
server resolves the bundled binary first — it sits next to the server on disk,
so it's found in both the terminal CLI and the Claude desktop app without
relying on your shell PATH. Confirm it's reachable:

```sh
"${CLAUDE_PLUGIN_ROOT:-.}/bin/wacli" --version 2>/dev/null || wacli --version 2>/dev/null || echo MISSING
```

A version number → done, skip to Step 2.

**Linux / Windows (or `MISSING` on macOS):** no bundled binary for your
platform — install wacli yourself:

```sh
brew install openclaw/tap/wacli
```

No `brew`? Install Homebrew first
(`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`),
or download the matching binary from the
[latest release](https://github.com/openclaw/wacli/releases) and put `wacli`
on PATH. Confirm `wacli --version` prints a version.

> Resolution order: `WACLI_PATH` → bundled `bin/darwin/wacli` (macOS) →
> `wacli` on PATH → `/opt/homebrew/bin` → `/usr/local/bin` →
> `/home/linuxbrew/.linuxbrew/bin`.

## Step 2 — the bun runtime

The MCP server is a bun script.

```sh
bun --version || "$HOME/.bun/bin/bun" --version
```

Missing → `curl -fsSL https://bun.sh/install | bash`. The plugin's `start`
script runs `bun install` itself on first launch, so there's nothing to
install by hand.

## Step 3 — link the device (a pairing code, in chat — no terminal, no QR)

This is the one real human step, and it happens **right here in the
conversation** using the **`authenticate` MCP tool**. Drive it for them:

1. **Ask for their WhatsApp phone number** in `+country` format (e.g.
   `+15551234567`) — the number whose WhatsApp account you're linking.
2. **Call the `authenticate` tool** with that number. It returns an
   8-character **pairing code** (shown like `ABCD-1234`).
3. **Relay the code** with these exact phone steps:
   > On your phone, open **WhatsApp → Settings → Linked Devices → Link a
   > Device**, tap **"Link with phone number instead"**, and enter the code:
   > **ABCD-1234**
4. **Wait, then confirm.** After they say they've entered it (or give it a few
   seconds), **call the `status` tool** and check for "device linked". Poll
   `status` every ~10s until it flips to linked. On link, wacli **automatically
   pulls in their recent messages** — `status` starts showing synced chats.

If the code lapses before they enter it (it's valid a few minutes), just call
`authenticate` again to issue a fresh one. If `authenticate` says it's already
linked, you're done — skip to Step 4.

> The session persists in wacli's local store (`~/.wacli`) and lasts as long as
> the device stays linked on the phone. Unlinking it from the phone ends the
> session and needs a fresh code. There is **no** Full Disk Access, SIP, or
> Automation prompt — device-linking is the whole permission model.

## Step 4 — first send + read

Have the user try a send to themselves: "WhatsApp me a hello" (to their own
number). A normal Claude permission prompt appears for the send — that's the
review-before-send flow working as intended. The message goes out as written.

Then a read: "what are my recent WhatsApp messages?" → `read_messages` should
return recent threads. (Reads refresh wacli's local store first, so the very
latest messages may take a sync pass to appear.)

## Optional — the AI-disclosure signature

By default, messages go out **as-is**: no "- Sent by Claude" stamp. The
signature is opt-in, and on WhatsApp it's worth leaving off — a visible
automation marker on every message is the kind of thing that gets numbers
flagged. If the user still wants machine-sent messages disclosed, mention they
can enable it:

- Globally: set `signature: true` in `~/.claude/whatsapp/config.json` (and
  optionally `signatureName`); or env `WHATSAPP_APPEND_SIGNATURE=true`.
- Per send: "send this as Acme" → the signature is added for that message only.

Once on, it appends `- Sent by Claude for <name>` (name = sign_as > config
`signatureName` > OS account name).

## Verify

Quickest end-to-end check is the status tool:

```
run /whatsapp-status
```

It reports wacli present, the device linked, and the local store reachable.

## Keeping the store warm (optional)

Reads auto-sync, so you don't need this. But if the user wants the freshest
possible reads or plans to use `/whatsapp-listen` heavily, they can run a
continuous sync in a spare terminal:

```sh
wacli sync --follow
```

The plugin coexists with it (wacli shares one store lock). Stop it with
Ctrl+C. To bound local disk growth: `wacli sync --follow --max-db-size 2GB`.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| tools say `wacli engine not found` | macOS: bundled (`bin/darwin/wacli`) — re-check Step 1. Linux/Windows: `brew install openclaw/tap/wacli` or set `WACLI_PATH` |
| no send/read tools in session | session predates the plugin enable — restart the session |
| tools error `not linked` / `pair` | Step 3 — call `authenticate` with their number, relay the code, poll `status` |
| `authenticate` won't return a code | usually a bad/unreachable number — re-confirm it in `+country` format and retry. If the store `lock` is held by a running `sync --follow`, stop that first |
| reads return nothing / look stale | the store hasn't synced yet — run `wacli sync --once`, or leave `wacli sync --follow` running |
| error mentions the store `lock` | another wacli process (often `sync --follow`) holds it — stop it or let it release, then retry |
| sends warn about rate / too fast | you're sending too quickly — slow down; WhatsApp ban-risk is real on a linked device |

### Deep fallback — manual terminal link (only if `authenticate` can't be used)

The in-chat `authenticate` flow above is the supported path. If for some reason
the tool isn't available (e.g. the MCP server isn't running) and the user *does*
have terminal access, they can link directly:

```sh
wacli auth --phone "+15551234567"   # prints a pairing code, same phone steps as Step 3
# or, if they prefer scanning:
wacli auth                          # prints a QR to scan
wacli auth status                   # verify; Ctrl+C once synced
```

This is a last resort — prefer the conversational flow for non-technical users.

## Auto-reply mode (not installed here)

If you later want Claude to *answer* messages unattended, that's a different
architecture (an always-on headless process driven by `wacli sync --follow`)
with real security **and** account-ban tradeoffs. See
[docs/AUTOREPLY.md](../../docs/AUTOREPLY.md) before offering it.
