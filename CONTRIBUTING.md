# Contributing

Thanks for your interest — contributions are welcome and encouraged. Issues
and PRs of any size help, whether it's a bug report, a docs fix, or wrapping
another piece of the wacli surface.

## How it's put together

This plugin is a **thin MCP server over [wacli](https://github.com/openclaw/wacli)**
(the WhatsApp engine — Go, MIT, built on whatsmeow). The real work happens in
wacli; this repo just exposes it as Claude tools and skills.

- `server.ts` — the MCP server. Each tool shells out to `wacli <cmd> --json`
  and renders the result. New tools should stay thin: drive wacli, don't
  reimplement WhatsApp logic here.
- `skills/` — the user-facing skills (setup, send, messages, listen, status).
- `docs/AUTOREPLY.md` — the unattended-reply recipe and its caveats.

## Dev setup

```sh
brew install openclaw/tap/wacli   # the engine
bun install                       # server deps
bunx tsc --noEmit                 # typecheck
```

A quick local smoke test (boots the server and lists tools) doesn't need a
paired account; anything that actually sends/reads needs `wacli auth` first.

## Guidelines

- **Keep the wrapper thin.** If wacli can do it, call wacli — prefer a new
  flag pass-through over new logic in `server.ts`.
- **Keep the AI disclosure intact.** Every send is signed
  `- Sent by Claude for <name>`; that line isn't optional, only the name
  varies. Don't add a path that sends unsigned.
- **Respect the human-in-the-loop posture.** No auto-reply in the default
  surface; reads run in wacli's `--read-only` mode.
- **Mind the account-ban reality.** WhatsApp linked-device automation carries
  real risk on a user's own number — don't add anything that encourages bulk
  or unattended blasting.
- Run `claude plugin validate .` and `bunx tsc --noEmit` before opening a PR.

Not sure where something fits? Open an issue first and we'll figure it out
together.
