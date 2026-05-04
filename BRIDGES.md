# Supported agent bridges

`m5stack-desktop-buddy` is bridge-agnostic — the firmware just reads a
generic heartbeat/prompt/turn JSON feed over BLE Nordic UART. A "bridge"
is any program that translates a coding agent's activity into that feed.
See [REFERENCE.md](REFERENCE.md) for the wire protocol.

This file tracks which bridges exist, what shape they take (first-party
desktop app, community CLI, user-hosted daemon…), and how far along each
one is.

## Status legend

- ✅ **shipped** — usable today
- 🚧 **in progress** — code exists, not yet feature-complete
- 💡 **proposed** — wanted, no code yet; PRs welcome

## Bridges

| Agent           | Bridge form                                  | Status        | Notes                                                              |
| --------------- | -------------------------------------------- | ------------- | ------------------------------------------------------------------ |
| Claude          | Claude desktop app (macOS / Windows)         | ✅ shipped    | Developer Mode → Open Hardware Buddy…; see [README.md](README.md). |
| Claude Code CLI | Companion daemon + Claude Code hook scripts  | ✅ shipped    | [`cc-buddy-bridge`](https://github.com/lu486/cc-buddy-bridge) — Python + `bleak`; round-trips PreToolUse permissions to the device. |
| OpenAI Codex CLI| Companion process watches `codex` log stream | 💡 proposed   | Needs stable event hooks from Codex CLI.                           |
| Cursor          | VS Code–style IDE listener                   | 💡 proposed   | Would expose agent runs happening inside Cursor.                   |
| Cline           | VS Code extension side-channel               | 💡 proposed   | Extension API exposes run state.                                   |
| Aider           | Stdin/stdout listener around `aider`         | 💡 proposed   | Easy target — aider is a well-behaved CLI.                         |
| Gemini CLI      | Companion process                            | 💡 proposed   | —                                                                  |
| Generic         | Custom bridge you write                      | ✅ always     | See "Writing a new bridge" below.                                  |

The table is intentionally open-ended. "Bridge" isn't a certification —
anything that can produce the JSON is a bridge.

## What a bridge has to do

A minimum-viable bridge:

1. **Discovers a device.** Scan for the Nordic UART Service UUID, pair,
   bond (recommended).
2. **Sends a heartbeat snapshot** on change and every ~10s as a
   keepalive. Required fields: `total`, `running`, `waiting`, `msg`.
   Everything else is optional but improves fidelity.
3. **Forwards permission prompts** by setting `prompt.{id,tool,hint}` on
   the snapshot when the agent blocks for approval.
4. **Handles permission acks** — listens for
   `{"cmd":"permission","id":...,"decision":"once"|"deny"}` and applies
   that decision to the underlying agent.
5. **Responds to status polls** — acks `{"cmd":"status"}` with whatever
   `bat`/`sys`/`stats` data the device publishes so the bridge can
   render a health panel. (Optional — the device is fine without a UI
   on the bridge side.)

A richer bridge additionally:

- Sends `turn` events for completed messages (text + tool-use content).
- Implements the folder-push transport so users can drop in GIF
  character packs.
- Supports the `name` / `owner` / `unpair` commands.

See [REFERENCE.md](REFERENCE.md) for field-level details.

## Writing a new bridge

The path of least resistance:

1. Read [REFERENCE.md](REFERENCE.md) end-to-end.
2. Build the smallest thing that can emit a keepalive snapshot — on
   macOS, `IOBluetooth` or `CoreBluetooth`; on Linux, `bluez` via
   `bleak` or `dbus`; on Node, `@abandonware/noble`. A ~200-line
   prototype in Python with `bleak` is enough to prove the device
   lights up.
3. Hook it into your agent's event stream. Most coding agents either
   emit structured logs, expose a local websocket, or have an SDK with
   "on tool call", "on message", "on approval" callbacks. Map those
   into the heartbeat shape.
4. Add permission routing last. Prompts are the interactive payoff —
   getting approvals to round-trip back into the agent is usually the
   hardest part and depends on whether the agent supports external
   decision hooks.

If you ship a bridge — first-party or community — open a PR adding a
row to the table above with a link. We'll keep the list honest about
status even if the code lives in your repo, not this one.

## A note on naming

The firmware currently advertises as `Claude-XXXX` for compatibility
with the Claude desktop app's device filter. When more bridges show up,
we'll either:

- switch to a bridge-neutral prefix like `Buddy-XXXX` and have bridges
  filter by NUS service UUID, or
- make the advertised prefix a device setting so users pick the
  namespace their bridge expects.

If you're writing a bridge today, filter by **service UUID** (stable)
rather than **name prefix** (will change).
