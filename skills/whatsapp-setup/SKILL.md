---
name: whatsapp-setup
description: One-time setup for WhatsApp from Claude — installs the wacli engine (official release download, checksum-verified) and links the device by scanning an in-chat QR (or entering a pairing code), no terminal, via the authenticate tool, then the initial sync. Use when the user asks to set up WhatsApp, when the WhatsApp tools (send_message/read_messages) are missing or erroring, or before the first send/read on a new machine.
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
a terminal**. Linking happens **right here in chat** — no terminal. Run every
check yourself; the only thing they do by hand is scan a QR with their phone
(or type one short code, if they prefer).

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

## Step 1 — install the wacli engine (you do this, via Bash)

The engine is the official binary from the
[openclaw/wacli](https://github.com/openclaw/wacli) GitHub release. You
download and install it — the user does nothing and needs no Homebrew or
developer tools; macOS's built-in `curl`/`tar` do the work. First check
whether it's already there:

```sh
"$HOME/.claude/whatsapp/engine/wacli" --version 2>/dev/null || wacli --version 2>/dev/null || echo MISSING
```

A version number → done, skip to Step 2.

**`MISSING` on macOS** → install it (pinned version, checksum-verified; the
universal binary covers Apple Silicon and Intel):

```sh
set -e
WACLI_VERSION="0.11.1"
WACLI_SHA256="8500eec89157356f16bb89160f91302d5134c068f791ae2dfea513b765016bec"
TMP="$(mktemp -d)"
curl -fsSL -o "$TMP/wacli.tar.gz" \
  "https://github.com/openclaw/wacli/releases/download/v$WACLI_VERSION/wacli_${WACLI_VERSION}_darwin_universal.tar.gz"
echo "$WACLI_SHA256  $TMP/wacli.tar.gz" | shasum -a 256 -c -
mkdir -p "$HOME/.claude/whatsapp/engine"
tar -xzf "$TMP/wacli.tar.gz" -C "$HOME/.claude/whatsapp/engine" wacli
rm -rf "$TMP"
"$HOME/.claude/whatsapp/engine/wacli" --version
```

A version number at the end → installed. If the checksum line fails, **stop**
— do not install the file — and re-download once; persistent mismatch means
the download is corrupted or tampered with, report it instead of proceeding.
(The pinned hash matches the `checksums.txt` published with the release.)

**Linux / Windows:** grab the matching archive from the
[latest release](https://github.com/openclaw/wacli/releases) the same way
(verify against its `checksums.txt`), or `brew install openclaw/tap/wacli`,
and confirm `wacli --version` prints a version.

> Resolution order: `WACLI_PATH` → `~/.claude/whatsapp/engine/wacli` →
> `wacli` on PATH → `/opt/homebrew/bin` → `/usr/local/bin` →
> `/home/linuxbrew/.linuxbrew/bin`. Power-user alternative:
> `brew install openclaw/tap/wacli`.

## Step 2 — the bun runtime

The MCP server is a bun script.

```sh
bun --version || "$HOME/.bun/bin/bun" --version
```

Missing → `curl -fsSL https://bun.sh/install | bash`. The plugin's `start`
script runs `bun install` itself on first launch, so there's nothing to
install by hand.

## Step 3 — link the device (scan a QR, in chat — no terminal)

This is the one real human step, and it happens **right here in the
conversation** using the **`authenticate` MCP tool**. Drive it for them:

1. **Call the `authenticate` tool** (no arguments — it defaults to the QR
   method). It renders a QR code to an image and **opens it on their screen**,
   and returns a file path as a backup.
2. **Walk them through scanning**, in plain language:
   > A QR code just opened on your screen. On your phone, open **WhatsApp →
   > Settings → Linked Devices → Link a Device**, and point the camera at it.
   >
   > (If nothing opened, open the image file I gave you and scan that.)
3. **Wait, then confirm.** After they say they've scanned it (or give it a few
   seconds), **call the `status` tool** and check for "device linked". Poll
   `status` every ~10s until it flips to linked. On link, wacli **automatically
   pulls in their recent messages** — `status` starts showing synced chats.

The QR refreshes about every 20 seconds. If it expires before they scan, just
call `authenticate` again for a fresh one. If `authenticate` says it's already
linked, you're done — skip to Step 4.

**Prefer a code over scanning?** If they'd rather type a code than scan (or the
QR won't open on their setup), call `authenticate` with `method: "code"` and
their **phone number** (`+country` format). It returns an 8-char code; they
enter it on the phone via **Linked Devices → Link a Device → "Link with phone
number instead"**. Same background-link + `status` poll as above.

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

## Optional — the hard approval gate

By default, sends go through the normal review-before-send flow above. For
users who want a **hard stop** — every outgoing message pauses on a yes/no
dialog that only a human can answer, shown in the client with the exact
recipient and text — offer the approval gate, especially on managed or
shared machines where "Claude must never send without my explicit OK" is a
requirement:

- Enable: set `approval: true` in `~/.claude/whatsapp/config.json` (create
  the file/dir if absent; read first and preserve other fields; 2-space
  indent). Applies from the next send — no restart needed.
- Explain the behavior in plain language: a dialog appears before each
  send/reaction; **Decline** cancels it and nothing goes out.
- Caveat: the dialog needs a client that supports approval prompts (MCP
  elicitation — Claude Code does). In a client that can't show them, the
  gate **blocks sends entirely rather than sending unapproved** — that's
  intentional (fail-closed), and turning the gate off is the only way to
  send from such a client.

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
| tools say `wacli engine not found` | Step 1 wasn't run (or the engine dir was deleted) — run the Step 1 install. Fallback: `brew install openclaw/tap/wacli` or set `WACLI_PATH` |
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
