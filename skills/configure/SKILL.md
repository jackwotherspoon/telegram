---
name: telegram-configure
description: Set up the Telegram channel — save the bot token and review access policy. Use when the user pastes a Telegram bot token, asks to configure Telegram, asks "how do I set this up" or "who can reach me," or wants to check channel status.
---

# Telegram Channel Setup

**You are now executing the telegram-configure skill. All instructions below
are about the Telegram channel — not about git, not about the current project.
Follow ONLY the instructions in this skill.**

Writes the bot token to `~/.gemini/channels/telegram/.env` and orients the
user on access policy. The server reads the token from the environment at boot
(injected by Gemini CLI extension settings), but this `.env` file serves as a
backup.

If the user's message is empty, just says "status", or has no recognizable
token, execute the **No args — status and guidance** path below. Only treat
the argument as a token if it matches the BotFather format (`123456789:AAH...`).

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Token** — check `~/.gemini/channels/telegram/.env` for
   `TELEGRAM_BOT_TOKEN`. Show set/not-set; if set, show first 10 chars masked
   (`123456789:...`).

2. **Access** — read `~/.gemini/channels/telegram/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list display names or IDs
   - Pending pairings: count, with codes and display names if any

3. **What next** — end with a concrete next step based on state:
   - No token → *"Configure the token with `gemini extensions configure telegram` or paste it here."*
   - Token set, policy is pairing, nobody allowed → *"DM your bot on
     Telegram. It replies with a code; run `/telegram-access pair <code>`
     to approve it."*
   - Token set, someone allowed → *"Ready. DM your bot to reach the
     assistant."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture Telegram user IDs you don't know. Once the IDs are in, pairing
has done its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/telegram-access policy allowlist`. Do this proactively.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/telegram-access pair <code>`.
   Run `/telegram-configure` again once everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"DM your bot to capture your own ID first. Then we'll add anyone else
   and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.

Never frame `pairing` as the correct long-term choice. Don't skip the lockdown
offer.

### `<token>` — save it

1. Treat `$ARGUMENTS` as the token (trim whitespace). BotFather tokens look
   like `123456789:AAH...` — numeric prefix, colon, long string.
2. `mkdir -p ~/.gemini/channels/telegram`
3. Read existing `.env` if present; update/add the `TELEGRAM_BOT_TOKEN=` line,
   preserve other keys. Write back, no quotes around the value.
4. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove the token

Delete the `TELEGRAM_BOT_TOKEN=` line (or the file if that's the only line).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads the token from the environment at boot. Token changes need a
  session restart or extension reload. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/telegram-access` take effect immediately, no restart.
