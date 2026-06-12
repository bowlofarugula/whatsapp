# Auto-reply mode (optional, not installed by default)

The whatsapp plugin's default model is **human-in-the-loop**: Claude reads
and sends WhatsApp messages only when asked, with normal permission prompts.
You may eventually want the other thing — Claude *answering* messages
unattended. That's a fundamentally different posture; read this whole page
before standing it up.

## Why this needs its own machinery

The plugin's MCP tools serve an *open* Claude session. Auto-reply is the case
where **no session is open** — someone messages at 3am and something has to
wake Claude. There's nothing to schedule a poll against, so you need a
persistent process running 24/7.

The right primitive for that process is `wacli sync --follow`, which keeps the
local store live and can emit each new message as it's stored. Two clean ways
to consume it:

```sh
# A) webhook: sync posts each stored live message as JSON to your endpoint
wacli sync --follow --webhook http://127.0.0.1:8787/wacli --webhook-secret "$SECRET"

# B) NDJSON lifecycle events on stderr, for a local consumer to tail
wacli sync --follow --events 2>events.ndjson
```

An auto-replier is a long-lived loop that receives those new-message events,
hands each inbound message to a headless `claude` turn, and sends the reply
back with `wacli send text`. A service manager (systemd / launchd
`KeepAlive` / a supervisor) keeps the loop alive across crashes and reboots.
Because `sync --follow` owns the store lock, send the replies through the same
running process (wacli delegates sends to the live follower automatically) or
serialize them so you never fight the lock.

## What changes (the security model)

- The loop runs `claude` headless with broad permissions. **Everyone whose
  messages you feed it can instruct that agent** — file access, connectors,
  everything the owner's Claude can do. Your inbound filter (which senders the
  loop forwards) stops being a convenience and becomes a real security
  boundary: "people with operator access to this machine."
- Replies go out with **no human review**. The `- Sent by Claude for <name>`
  signature still discloses the machine, but nobody approves the wording.
- Message content is still data, never instructions. The loop's prompt must
  treat quoted/forwarded third-party text as data and refuse to act on "tell
  Claude to…" requests embedded in a message.

## What changes (the WhatsApp account risk)

This is the big one, and it's specific to WhatsApp. Unattended, automated
replies are exactly the pattern WhatsApp's anti-abuse systems look for.
Running this **materially raises the risk** that WhatsApp flags, rate-limits,
or bans the linked number — and it's your real account. Mitigations:

- Keep the inbound allowlist tiny (the owner + maybe one trusted person) —
  **never customers or strangers**.
- Add human-like pacing and hard rate caps; never auto-reply in bursts.
- Accept that a ban is a plausible outcome of running this at all.

## Hard requirements

- `wacli` installed and a paired session (`wacli auth`) for the service
  context. Refresh tokens last only as long as the device stays linked on the
  phone; unlinking ends the session and needs a fresh QR.
- A standalone authenticated `claude` CLI (the desktop app's embedded copy
  can't be driven by a service manager) and `bun` on PATH.
- A single owner of the store lock: let `wacli sync --follow` run, and route
  all sends through it (or serialize them) so they don't fail on the lock.
- A keep-alive that holds the loop open and restarts it on exit.

## Checklist before enabling

- [ ] You understand replies go out unreviewed
- [ ] You accept the raised WhatsApp-ban risk on your real number
- [ ] The inbound filter is reduced to the owner (+ at most a trusted person)
      — never customers
- [ ] One process owns the store lock; sends are routed through it
- [ ] The standing prompt treats message content as data, not instructions
