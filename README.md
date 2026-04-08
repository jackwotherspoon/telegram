# Telegram Channel for Gemini CLI

> [!NOTE]
> This implementation is inspired by and based on the [Claude Code Telegram Plugin](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram).

Bridge your Telegram bot to a running Gemini CLI session — messages arrive in real time, and the assistant replies through the bot.

## Quick Start

### 1. Create a bot

Open Telegram, DM [@BotFather](https://t.me/BotFather), and send `/newbot`. Follow the prompts. Copy the token it gives you — it looks like `123456789:AAH...`.

### 2. Install the extension

```bash
gemini extensions install https://github.com/jackwotherspoon/telegram
```

You will be prompted for your bot token during installation:

```
? Telegram Bot Token
Bot token from @BotFather on Telegram ›
```

Paste your bot token. It's stored securely in your system keychain.
### 3. Start Gemini CLI with the channel enabled

```bash
gemini --channels telegram
```

The bot starts polling automatically. You should see `» Channels listening for messages: telegram` in the Gemini CLI session.

### 4. Verify status

Inside Gemini CLI, you can run `/channels` to see all active message channels and their status.

### 5. Pair yourself
DM your bot on Telegram. It replies with a 6-character pairing code:

```
Pairing required — ask the user in Gemini CLI to run:

/telegram-access pair a1b2c3
```

Run that command in Gemini CLI. The bot confirms on Telegram: **"Paired! Say hi to Gemini."**

### 6. Send a message

DM the bot again. Your message appears in the Gemini CLI session and the assistant responds through the bot.

---

## Access Control

Every inbound message passes through a gate that checks the current DM policy. There are three modes:

| Policy | Who gets through | Use case |
|--------|-----------------|----------|
| `pairing` | Approved users + unknown users receive a pairing code | Onboarding — capturing new user IDs |
| `allowlist` | Only approved users; everyone else is silently dropped | Production — locked down |
| `disabled` | Nobody, not even approved users | Maintenance or temporary shutdown |

### Recommended workflow

1. **Start in `pairing` mode** (the default). DM the bot to capture your own ID.
2. **Approve yourself** with `/telegram-access pair <code>`.
3. **Have others DM the bot** if they need access. Approve each one.
4. **Lock it down** once everyone's in:

```
/telegram-access policy allowlist
```

> `pairing` mode is for setup, not day-to-day use. Once your allowlist is populated, switch to `allowlist` so unknown users can't trigger pairing codes.

---

## Commands

### `/telegram-configure`

| Usage | Description |
|-------|-------------|
| `/telegram-configure status` | Show token (masked), access policy, allowlist, and next steps |
| `/telegram-configure <token>` | Save a bot token to the backup `.env` file |
| `/telegram-configure clear` | Remove the saved token |

> The primary token is stored in your system keychain via extension settings. The `.env` file is a backup the server reads at boot if the environment variable is missing.

### `/telegram-access`

| Usage | Description |
|-------|-------------|
| `/telegram-access status` | Show current policy, allowlist, and pending pairings |
| `/telegram-access pair <code>` | Approve a pending pairing request |
| `/telegram-access deny <code>` | Reject a pending pairing request |
| `/telegram-access allow <userId>` | Add a user to the allowlist directly (no pairing needed) |
| `/telegram-access remove <userId>` | Remove a user from the allowlist |
| `/telegram-access policy <mode>` | Set DM policy: `pairing`, `allowlist`, or `disabled` |

---

## Group Chats

The bot can participate in Telegram group chats with per-group configuration.

```
/telegram-access group add <groupId>
```

By default, the bot only responds when mentioned (e.g., `@yourbotname what's the status?`). To disable the mention requirement:

```
/telegram-access group add <groupId> --no-mention
```

To restrict which group members can trigger the bot:

```
/telegram-access group add <groupId> --allow 123456,789012
```

Remove a group:

```
/telegram-access group rm <groupId>
```

> To find a group's ID, add the bot to the group and check the MCP server logs, or use Telegram API tools.

---

## What the Assistant Can Do

| Tool | Description |
|------|-------------|
| `reply` | Send a text message. Supports `reply_to` for threading and `files` for attachments (absolute paths). |
| `react` | Add an emoji reaction to a message. Limited to Telegram's built-in emoji whitelist. |
| `edit_message` | Update a previously sent message — useful for replacing "working..." with a final result. |

The assistant can also receive and view photos sent to the bot. Photos are downloaded to a local cache and passed as an `image_path` attribute.

---

## Configuration

Fine-tune delivery behavior with `/telegram-access set <key> <value>`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `ackReaction` | emoji string | `""` | React to inbound messages with this emoji. Empty string disables. |
| `replyToMode` | `off` / `first` / `all` | `off` | Whether to use Telegram reply-threading on outbound messages. |
| `textChunkLimit` | number | `4096` | Maximum characters per message before splitting. |
| `chunkMode` | `length` / `newline` | `length` | Split long messages by character count or at newline boundaries. |
| `mentionPatterns` | JSON array | `["@botname"]` | Regex patterns that trigger the bot in group chats. |

---

## Limitations

- **No message history.** Telegram's Bot API has no endpoint for fetching past messages or searching. The assistant only sees messages as they arrive. If earlier context is needed, paste or summarize it.
- **Token changes require a restart.** The bot token is read at boot. After changing it with `gemini extensions configure telegram`, restart Gemini CLI.
- **Access changes are immediate.** Edits to the allowlist or DM policy via `/telegram-access` take effect on the next inbound message — no restart needed.

---

## Troubleshooting

**Bot doesn't respond to DMs**
- Check that you started Gemini with `--channels telegram`.
- Verify the token is configured: `/telegram-configure status`.
- Check MCP server status — the telegram server should show as connected.

**Messages don't appear in Gemini CLI**
- Confirm the sender is on the allowlist or that the DM policy is `pairing`.
- Check that the channel is active by running `/channels` in Gemini CLI.

**Pairing code not working**
- Codes expire after 10 minutes. Ask the user to DM the bot again for a fresh code.
- Make sure you're copying the exact 6-character code.

**"Chat is not allowlisted" error when replying**
- The assistant can only reply to chats with approved senders. Add the user with `/telegram-access allow <userId>` or approve their pairing code.
