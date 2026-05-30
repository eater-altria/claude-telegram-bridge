# claude-telegram-bridge

A Claude Code plugin marketplace hosting **`telegram`** — a fork of the official
[Telegram plugin](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram)
with three added behaviors:

1. **Persistent typing indicator** — the bot shows "typing…" continuously from
   the moment your message reaches Claude until it responds, instead of the ~5s a
   single Telegram typing action lasts.
2. **👀 acknowledgement reaction** — every delivered message gets a 👀 so you can
   see it was received (configurable / disablable).
3. **Command menu + local-skill discovery** — built-in Claude commands plus your
   own local skills/commands (from `~/.claude` and the project's `.claude`) are
   auto-registered into Telegram's "/" autocomplete.

Full plugin docs: [`telegram/README.md`](./telegram/README.md) ·
access control: [`telegram/ACCESS.md`](./telegram/ACCESS.md).

## Install

```
/plugin marketplace add eater-altria/claude-telegram-bridge
/plugin install telegram@claude-telegram-bridge
/reload-plugins
```

Then give the server your bot token and relaunch with the channel flag:

```
/telegram:configure <BOT_TOKEN>
```
```sh
claude --channels plugin:telegram@claude-telegram-bridge
```

DM the bot, then pair from your session:

```
/telegram:access pair <code>
```

See [`telegram/README.md`](./telegram/README.md) for the full setup, photos,
access policies, and the tools the bot exposes to the assistant.

## Updating

After new commits land here:

```
/plugin marketplace update claude-telegram-bridge
/plugin install telegram@claude-telegram-bridge
```

## License

Apache-2.0 (inherited from the upstream plugin). See [LICENSE](./LICENSE).
