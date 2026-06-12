---
name: whatsapp-status
description: Check that WhatsApp from Claude is healthy — wacli engine installed, device linked, local store reachable, tools live. Use when the user asks whether WhatsApp works, why a send/read failed, or who Claude can message.
user-invocable: true
---

# /whatsapp-status — WhatsApp health check

Run all checks, then report in plain language: one line per check with ✅/❌,
then a "what to do" section only if something failed (point at the matching
`/whatsapp-setup` step).

1. **wacli engine present**: /whatsapp-setup installs it to `~/.claude/whatsapp/engine` —
   `"${CLAUDE_PLUGIN_ROOT:-.}/bin/wacli" --version 2>/dev/null || wacli --version 2>/dev/null || echo MISSING`.
   `MISSING` → setup step 1 installs it (fallback `brew install openclaw/tap/wacli` or set `WACLI_PATH`).
2. **bun present**: `bun --version || "$HOME/.bun/bin/bun" --version`.
   Missing → setup step 2.
3. **Device linked**: call the `status` tool (or `wacli auth status`).
   Not authenticated → run `/whatsapp-setup` (or the `authenticate` tool): it
   pops up a QR for them to scan with their phone (or `method:"code"` + their
   number for a pairing code) — no terminal.
4. **Local store reachable**: `wacli doctor --json 2>&1 | head -40` — reports
   store layout, auth identity, FTS/search state, and counts. A quick content
   check: `wacli chats list --limit 1 --json --read-only` returns a chat (or an
   empty list if nothing has synced yet → suggest `wacli sync --once`).
5. **MCP tools live**: check whether the `send_message` / `list_chats` /
   `read_messages` / `search_messages` / `react` / `watch` tools from this
   plugin's WhatsApp server are available in this session (load via ToolSearch
   if deferred). If present, the fastest single check is to **call the
   `status` tool** — it confirms wacli installed, the device linked, and the
   store reachable in one shot. Missing while 1–4 pass → the session predates
   the plugin; restart it.

If everything passes, suggest: "WhatsApp yourself a hello, then ask me for
your recent messages."

Remind the user once, if relevant: this is their real WhatsApp account on a
linked device — keep sends human-paced to avoid rate-limits/bans.
