<p align="center">
  <h1 align="center">🔀 Telegram Bot-to-Bot Bridge</h1>
  <p align="center">
    <strong>Break Telegram's bot-to-bot limitation. Let your bots talk to each other.</strong>
  </p>
  <p align="center">
    <a href="#quick-start">Quick Start</a> •
    <a href="#how-it-works">How It Works</a> •
    <a href="#configuration">Configuration</a> •
    <a href="#troubleshooting">Troubleshooting</a>
  </p>
  <p align="center">
    <img src="https://img.shields.io/badge/python-3.8+-blue.svg" alt="Python 3.8+">
    <img src="https://img.shields.io/badge/telegram-MTProto-blue.svg" alt="Telegram MTProto">
    <img src="https://img.shields.io/badge/telethon-latest-green.svg" alt="Telethon">
    <img src="https://img.shields.io/badge/license-MIT-green.svg" alt="MIT License">
    <img src="https://img.shields.io/badge/multi--group-✓-brightgreen.svg" alt="Multi-Group">
    <img src="https://img.shields.io/badge/multi--bot-✓-brightgreen.svg" alt="Multi-Bot">
    <img src="https://img.shields.io/badge/auto--delete-✓-brightgreen.svg" alt="Auto-Delete">
    <img src="https://img.shields.io/badge/zero--truncation-✓-brightgreen.svg" alt="Zero Truncation">
  </p>
</p>

---

## The Problem

Telegram's Bot API has a hard limitation: **bots cannot see messages from other bots** in groups. This is server-side and cannot be bypassed with privacy settings, admin status, or any configuration.

If you run multiple AI agents (OpenClaw, LangChain, AutoGPT, etc.) in a Telegram group, they're blind to each other.

## The Solution

This bridge uses a **MTProto user account** to relay messages between bots using `formatting_entities` — Telegram's native mention system. No HTML parsing, no content corruption, no truncation.

```
Bot A: "@BotB check this contract"
  ↓ instant
Bridge: sends relay (no mention — BotB ignores, humans see it)
  ↓ streaming edits mirrored
Bridge: edits stop → adds @BotB mention entity
  ↓ BotB's API sees it
BotB: responds
  ↓ 2 seconds
Bridge: auto-deletes relay (clean chat)
```

## Features

| Feature | Description |
|---------|-------------|
| 🔀 **Multi-Group** | Watch multiple Telegram groups simultaneously |
| 🤖 **Multi-Bot** | Relay between any number of bots |
| ⚡ **Instant Relay** | No debounce delay — relay appears immediately |
| 📝 **Edit Mirroring** | Streaming bot responses are mirrored in real-time |
| 🎯 **Smart Mention** | Uses `formatting_entities` — zero truncation, zero corruption |
| 🗑️ **Auto-Delete** | Relay messages vanish after the target bot receives them |
| 🔄 **FloodWait Handling** | Automatic retry with backoff on rate limits |
| 📋 **Relay Logging** | Saves all relayed messages to `relay/history.jsonl` |
| 🖥️ **systemd Ready** | Runs as a background service with auto-restart |

## How It Works

```
┌─────────────────────────────────────────────┐
│                    HOST                      │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │  Telegram Bot-to-Bot Bridge         │    │
│  │  (MTProto user session)             │    │
│  │                                      │    │
│  │  Watches: Group 1, Group 2, ...      │    │
│  │  Relays:  Bot A ↔ Bot B ↔ Bot C     │    │
│  │  Method:  formatting_entities        │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Bot A   │  │  Bot B   │  │  Bot C   │  │
│  │ (any fw) │  │ (any fw) │  │ (any fw) │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

The bridge works with **any bot framework** — OpenClaw, LangChain, AutoGPT, python-telegram-bot, Telegraf, or custom bots. It doesn't need access to the bots themselves, only a user account in the group.

## Quick Start

### 1. Install

```bash
git clone https://github.com/naorbrig/openclaw-telegram-bridge.git
cd openclaw-telegram-bridge
pip install telethon
```

### 2. Get Telegram API Credentials

1. Go to [my.telegram.org/apps](https://my.telegram.org/apps)
2. Create an application
3. Copy the `api_id` and `api_hash`

### 3. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```env
TG_API_ID=12345678
TG_API_HASH=abcdef1234567890abcdef1234567890

BRIDGE_GROUPS=1234567890,9876543210

BRIDGE_BOTS='[
  {"username": "my_first_bot", "alts": ["my_first_bot"], "mention": "@my_first_bot"},
  {"username": "my_second_bot", "alts": ["my_second_bot"], "mention": "@my_second_bot"}
]'
```

### 4. Run

```bash
python bridge.py
```

First run prompts for phone number authentication. Session is saved and reused.

### 5. Run as a Service (Recommended)

```bash
sudo cp bridge.service.example /etc/systemd/system/telegram-bridge.service
# Edit paths and user in the service file
sudo systemctl daemon-reload
sudo systemctl enable telegram-bridge
sudo systemctl start telegram-bridge
```

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TG_API_ID` | ✅ | — | Telegram API ID from [my.telegram.org](https://my.telegram.org/apps) |
| `TG_API_HASH` | ✅ | — | Telegram API hash |
| `TG_SESSION_NAME` | — | `bridge_session` | Session file name |
| `BRIDGE_GROUPS` | ✅ | — | Comma-separated MTProto channel IDs |
| `BRIDGE_BOTS` | ✅ | — | JSON array of bot configurations |
| `MENTION_SILENCE` | — | `5` | Seconds of no edits before adding mention |
| `DELETE_DELAY` | — | `2` | Seconds after mention before deleting relay |
| `EDIT_THROTTLE` | — | `1.5` | Min seconds between mirror edits |

### Bot Configuration

```json
[
  {
    "username": "my_bot",
    "alts": ["my_bot", "mybot"],
    "mention": "@my_bot"
  }
]
```

| Field | Required | Description |
|-------|----------|-------------|
| `username` | ✅ | Bot's Telegram username (lowercase, no @) |
| `alts` | — | Alternative spellings to match |
| `mention` | ✅ | Mention text (e.g., `@my_bot`) |

### Finding Group IDs

```bash
# Method 1: Bot API
curl "https://api.telegram.org/bot<TOKEN>/getUpdates" | jq '.result[].message.chat.id'
# Group ID: -1001234567890 → MTProto format: 1234567890

# Method 2: Check bridge logs after adding the bot and sending a message
```

## Why `formatting_entities`?

Previous approaches used `parse_mode="html"` with `<a href="tg://resolve?domain=bot">@bot</a>`. This breaks when bot messages contain `<`, `>`, `&`, or other HTML-special characters — causing **message truncation**.

`formatting_entities` attaches a structural `InputMessageEntityMentionName` to the message text. The text passes through **completely raw** — no escaping, no parsing, no corruption. Telegram's server recognizes the entity and delivers the mention notification to the target bot.

```python
# ❌ Old approach — HTML parsing corrupts content
await client.edit_message(entity, msg_id, html_text, parse_mode="html")

# ✅ This bridge — raw text + entity, zero corruption
mention = InputMessageEntityMentionName(offset=0, length=14, user_id=bot_entity)
await client.edit_message(entity, msg_id, raw_text, formatting_entities=[mention])
```

## Timing Diagram

```
t=0.0s  Bot A sends message (30 chars, streaming starts)
t=0.0s  Bridge sends relay instantly [Bot A]: message...
t=0.5s  Bot A edits (150 chars) → Bridge mirrors edit
t=1.0s  Bot A edits (400 chars) → Bridge mirrors edit
t=2.0s  Bot A edits (800 chars) → Bridge mirrors edit
t=3.0s  Bot A edits (1200 chars) → Bridge mirrors edit (throttled)
t=4.0s  Bot A finishes streaming (1500 chars)
t=9.0s  5s silence → Bridge adds @BotB mention entity
t=9.0s  BotB's Bot API receives edited_message with mention → BotB triggers
t=11.0s Bridge auto-deletes relay message
```

## Troubleshooting

<details>
<summary><b>Messages not being relayed</b></summary>

- Verify bot username in `BRIDGE_BOTS` matches exactly (lowercase)
- Verify group channel ID in `BRIDGE_GROUPS` (MTProto format, no `-100`)
- Verify the user account is a member of the group
- Check logs: `tail -f bridge.log` — look for `[NEW]` entries

</details>

<details>
<summary><b>FloodWait errors</b></summary>

Bridge handles these automatically. If frequent, increase `EDIT_THROTTLE`:
```env
EDIT_THROTTLE=3
```

</details>

<details>
<summary><b>Relay messages not being deleted</b></summary>

- The target bot may react before deletion — harmless cosmetic error
- Increase `DELETE_DELAY` if needed:
```env
DELETE_DELAY=4
```

</details>

<details>
<summary><b>Bot not responding to relayed messages</b></summary>

- Ensure the bot's username appears in the original message
- Check that `formatting_entities` are added: look for `MENTION added` in logs
- Verify the bot entity was resolved at startup: look for `Bot entity resolved`

</details>

<details>
<summary><b>Mention triggers on partial message (streaming)</b></summary>

Increase `MENTION_SILENCE` to wait longer for streaming to finish:
```env
MENTION_SILENCE=10
```

</details>

## Security

| Aspect | Detail |
|--------|--------|
| 🔒 **Session file** | Contains your Telegram account access — keep it secure |
| 🚫 **No bot tokens needed** | Bridge uses only the user account |
| 🗑️ **Auto-cleanup** | Relay messages are deleted automatically |
| 📁 **Host-only** | Runs on the host, not inside bot containers |
| 🔐 **Group restriction** | Only watches configured groups |

## Works With

- [OpenClaw](https://openclaw.ai) — AI agent deployment platform
- [LangChain](https://langchain.com) — LLM application framework
- [AutoGPT](https://autogpt.net) — Autonomous AI agents
- [python-telegram-bot](https://python-telegram-bot.org) — Python Telegram bot framework
- [Telegraf](https://telegraf.js.org) — Node.js Telegram bot framework
- Any bot that uses Telegram's Bot API

## Contributing

PRs welcome! Please ensure:
- No hardcoded values (use environment variables)
- No sensitive data in commits
- Test with at least 2 bots in a group before submitting

## License

MIT — use it however you want.

---

<p align="center">
  Built to solve a real problem. If this helped you, ⭐ the repo.
</p>
