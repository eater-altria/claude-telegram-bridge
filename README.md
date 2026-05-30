# Telegram

Connect a Telegram bot to your Claude Code with an MCP server.

The MCP server logs into Telegram as a bot and provides tools to Claude to reply, react, or edit messages. When you message the bot, the server forwards the message to your Claude Code session.

## Fork customizations

This fork adds three behaviors on top of the upstream plugin:

1. **Persistent typing indicator.** The bot shows "typing…" continuously from the moment your message is delivered to Claude until Claude responds — not just the ~5s a single Telegram typing action lasts. The keepalive (`startTyping`/`stopTyping`, ~4s refresh) stops as soon as Claude sends a `reply`, `react`, or `edit_message` to that chat, or when a permission request is raised (Claude is then blocked on you). A 5-minute hard cap bounds the rare turn that produces no outbound at all. *Known limits:* if Claude replies, keeps thinking, then replies again, typing is off during the gap (the server can't see "the turn is still going"); and rapid back-to-back messages in one chat share a single indicator slot. Both are bounded and harmless for a single-user bridge.
2. **👀 acknowledgement reaction.** Every inbound message that passes the gate and reaches Claude gets a 👀 reaction so you can see it was received. This is now the default; override it per-instance with `ackReaction` in `access.json` (set a different whitelisted emoji to change it, or `""` to disable).
3. **Claude Code command menu + local skill discovery.** `setMyCommands` registers, in Telegram's "/" autocomplete:
   - the local pairing commands `/start` `/help` `/status` (handled by the bot itself, never forwarded);
   - a curated set of built-in Claude commands (`/mcp`, `/context`, `/review`, `/code_review`, `/security_review`, `/simplify`, `/verify`, `/init`, `/deep_research`, `/recap`, `/btw`) — built-in skills aren't on disk, so these are hardcoded;
   - **your local skills and slash commands, auto-discovered at startup** by scanning `~/.claude/skills/*/SKILL.md`, `~/.claude/commands/**/*.md`, and the same under the project's `.claude` (located via the `CLAUDE_PROJECT_DIR` env var Claude Code injects). Each skill/command's `description:` frontmatter becomes the menu tooltip. So a personal skill like `cheap-coder` shows up as `/cheap_coder`.

   **All of these except `/start` `/help` `/status` are forwarded to the session as ordinary message text — they are NOT executed as terminal/harness slash commands.** Claude acts on the intent with the tools it has (e.g. it can invoke the matching skill via its Skill tool); it can't open harness UIs, switch models, manage MCP/OAuth, or read local billing, so commands that only make sense as TUI/harness actions (`/clear`, `/compact`, `/model`, `/config`, …) are deliberately left out of the curated set. Telegram command names must match `^[a-z0-9_]{1,32}$`, so `-`/`:` in names (e.g. `code-review`, `telegram:access`, namespaced commands) become `_`. The menu is capped at Telegram's 100-command limit.

   Config in `access.json`: `registerLocalSkills: false` disables the disk scan (curated + pairing commands still register); `extraSkillDirs: ["/path/to/.claude"]` adds extra roots to scan for `skills/` and `commands/`.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.

## Quick Setup
> Default pairing flow for a single-user DM bot. See [ACCESS.md](./ACCESS.md) for groups and multi-user setups.

**1. Create a bot with BotFather.**

Open a chat with [@BotFather](https://t.me/BotFather) on Telegram and send `/newbot`. BotFather asks for two things:

- **Name** — the display name shown in chat headers (anything, can contain spaces)
- **Username** — a unique handle ending in `bot` (e.g. `my_assistant_bot`). This becomes your bot's link: `t.me/my_assistant_bot`.

BotFather replies with a token that looks like `123456789:AAHfiqksKZ8...` — that's the whole token, copy it including the leading number and colon.

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

Install the plugin:
```
/plugin install telegram@claude-plugins-official
/reload-plugins
```

**3. Give the server the token.**

```
/telegram:configure 123456789:AAHfiqksKZ8...
```

Writes `TELEGRAM_BOT_TOKEN=...` to `~/.claude/channels/telegram/.env`. You can also write that file by hand, or set the variable in your shell environment — shell takes precedence.

> To run multiple bots on one machine (different tokens, separate allowlists), point `TELEGRAM_STATE_DIR` at a different directory per instance.

**4. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
claude --channels plugin:telegram@claude-plugins-official
```

**5. Pair.**

With Claude Code running from the previous step, DM your bot on Telegram — it replies with a 6-character pairing code. If the bot doesn't respond, make sure your session is running with `--channels`. In your Claude Code session:

```
/telegram:access pair <code>
```

Your next DM reaches the assistant.

> Unlike Discord, there's no server invite step — Telegram bots accept DMs immediately. Pairing handles the user-ID lookup so you never touch numeric IDs.

**6. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so strangers don't get pairing-code replies. Ask Claude to do it, or `/telegram:access policy allowlist` directly.

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, groups, mention detection, delivery config, skill commands, and the `access.json` schema.

Quick reference: IDs are **numeric user IDs** (get yours from [@userinfobot](https://t.me/userinfobot)). Default policy is `pairing`. `ackReaction` only accepts Telegram's fixed emoji whitelist.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading and `files` (absolute paths) for attachments. Images (`.jpg`/`.png`/`.gif`/`.webp`) send as photos with inline preview; other types send as documents. Max 50MB each. Auto-chunks text; files send as separate messages after the text. Returns the sent message ID(s). |
| `react` | Add an emoji reaction to a message by ID. **Only Telegram's fixed whitelist** is accepted (👍 👎 ❤ 🔥 👀 etc). |
| `edit_message` | Edit a message the bot previously sent. Useful for "working…" → result progress updates. Only works on the bot's own messages. |

Inbound messages trigger a typing indicator automatically — Telegram shows
"botname is typing…" while the assistant works on a response.

## Photos

Inbound photos are downloaded to `~/.claude/channels/telegram/inbox/` and the
local path is included in the `<channel>` notification so the assistant can
`Read` it. Telegram compresses photos — if you need the original file, send it
as a document instead (long-press → Send as File).

## No history or search

Telegram's Bot API exposes **neither** message history nor search. The bot
only sees messages as they arrive — no `fetch_messages` tool exists. If the
assistant needs earlier context, it will ask you to paste or summarize.

This also means there's no `download_attachment` tool for historical messages
— photos are downloaded eagerly on arrival since there's no way to fetch them
later.
