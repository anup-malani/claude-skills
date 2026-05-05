# Phone-Accessible Claude Code: Channels Setup

> **Status:** iMessage path tested and working. Telegram path is documented from the official plugin but the author has not yet personally walked through it end-to-end — treat it as beta. PRs and corrections welcome.

This pattern gives you phone access to one or more Claude Code assistants running on an always-on Mac at home. Each assistant lives in its own project directory and persistent tmux session. You reach them over iMessage (text yourself) or Telegram (one bot per project). Messages route to the right session, Claude processes them with full access to all local MCP servers and files, and replies come back to your phone in seconds. No persistent relay connection means hotel WiFi, app backgrounding, and network hiccups don't break anything mid-task. This is the setup described in the companion post on X ([@anup_malani](https://x.com/anup_malani), 2026-05-05).

---

## TL;DR — bootstrap prompt

Paste this into a fresh Claude Code terminal on your always-on Mac. It will walk you through the full setup interactively.

```
I want to set up Claude Code channels so I can reach my assistants from my phone.

My setup:
- Mac (always on — Mac Mini or similar)
- Claude Code installed (v2.1.80 or later required for channels)
- Bun installed at ~/.bun/bin/bun — if missing, install it first: curl -fsSL https://bun.sh/install | bash

Please do the following:

1. CHECK ENVIRONMENT
   - Confirm Claude Code version is v2.1.80+: run `claude --version`
   - Confirm Bun is available: run `bun --version` (or `~/.bun/bin/bun --version`)
   - List my current projects so we can decide which ones get a Telegram bot

2. iMESSAGE SETUP (Mac-only, no bot token needed)
   - Install the iMessage plugin: /plugin install imessage@claude-plugins-official
   - If not found, first run: /plugin marketplace add anthropics/claude-plugins-official
   - Then: /reload-plugins
   - Restart Claude Code with: claude --channels plugin:imessage@claude-plugins-official
   - Walk me through granting Full Disk Access for my terminal app if needed
   - Have me text myself to verify it works

3. TELEGRAM SETUP (one bot per project I specify)
   - Walk me through BotFather for each project:
     * Open https://t.me/BotFather in Telegram
     * Send /newbot
     * Give it a display name (e.g. "My Research Assistant")
     * Give it a username ending in "bot" — letters, digits, underscores only, 5-32 chars
       (Example pattern: yourproject_bot. Keep it short and opaque if you care about discretion.)
     * Copy the token BotFather returns — paste it here when prompted
   - I will create one bot per project and paste the tokens one at a time
   - For EACH project/bot:
     * Install the Telegram plugin: /plugin install telegram@claude-plugins-official
     * Configure the token: /telegram:configure <token>
       NOTE: this goes into the PROJECT's .claude/settings.local.json, NOT ~/.claude/settings.json
       so each session picks up its own bot. Do not write tokens to any checked-in file.
     * Set up a persistent tmux session for this project:
       tmux new-session -d -s <project-name> -c /path/to/project
     * Start Claude inside that pane with channels enabled:
       tmux send-keys -t <project-name> "claude --channels plugin:telegram@claude-plugins-official" Enter
     * Walk me through the pairing flow:
       - Send any message to the bot from my phone
       - Run: /telegram:access pair <code>
       - Lock down: /telegram:access policy allowlist

4. VERIFY
   - For each bot: send a test message from phone, confirm the right assistant responds
   - If anything fails, diagnose and fix before moving on

Defaults:
- Bun known-working version: 1.3.13 (use current stable if not already installed)
- Per-project token config: .claude/settings.local.json (gitignored)
- One bot per project — do not share tokens across sessions
- Do not advertise bot usernames publicly; use opaque names

Ask me for the list of projects before doing anything. Then proceed step by step.
```

---

## What you're building

```
Your phone
  │
  ├── iMessage (text yourself)
  │     └── iMessage plugin reads ~/Library/Messages/chat.db
  │           └── Delivers <channel> event to Claude Code session
  │                 └── Claude processes locally → AppleScript reply
  │
  └── Telegram (one bot per project)
        └── Telegram plugin polls bot API
              └── Delivers <channel> event to Claude Code session (project dir)
                    └── Claude processes locally with full MCP access → bot reply

Claude Code sessions run inside persistent tmux panes on your always-on Mac.
MCP servers (Gmail, Calendar, local tools) run locally — they never go through a relay.
```

**How this differs from Remote Control:**

Remote Control (RC) keeps a persistent relay connection between the Claude app on your phone and your local session. When the relay drops — network hiccup, app backgrounding, session timeout — you lose the session. Some MCP servers (HTTP servers with API keys) fail through the relay entirely.

Channels are async. Your phone sends a message; the plugin reads it; Claude processes it locally; the reply goes back. No persistent connection. If your network drops between messages, nothing is lost. The tradeoff: you cannot watch Claude think in real time or interrupt a running task. For task-based requests — "check my email," "run a skill," "create a calendar event" — channels are the more reliable choice.

---

## Why channels (vs Remote Control)

- **Reliability on bad networks.** No persistent relay means hotel WiFi drops don't kill your session mid-task.
- **MCP servers work normally.** Your local MCP servers (Gmail, Calendar, Typefully, custom tools) run in the session without going through any relay. RC sends traffic through Anthropic's relay infrastructure; some MCP servers return 401 errors through it.
- **One bot, one project.** With Telegram you can give each assistant its own identity. The right Claude session answers because it's pinned to the right project directory.

For more on the tradeoffs: [@anup_malani on X, 2026-05-05](https://x.com/anup_malani) (link TBD after post goes live).

---

## Prerequisites

- **Mac** running macOS (iMessage plugin is Mac-only; Telegram is cross-platform but this guide targets Mac)
- **Claude Code v2.1.80 or later** — channels require this minimum version and a claude.ai login (API key auth is not supported for channels)
  - Check: `claude --version`
  - Upgrade: `npm update -g @anthropic-ai/claude-code` or however you installed it
- **Bun** — required by all channel plugins (they are Bun scripts)
  - Check: `bun --version`
  - Install: `curl -fsSL https://bun.sh/install | bash`
  - Known-working version: **1.3.13** (use current stable)
- **An always-on device** — Mac Mini, MacBook that never sleeps, a VPS running macOS-compatible Claude Code. The session must be running for channels to deliver messages.
- **Telegram account** (for Telegram bots) — no special developer account needed; any Telegram account can create bots via BotFather.

---

## Setup, step by step

### 1. Install Bun

If `bun --version` returns a version, skip this step.

```bash
curl -fsSL https://bun.sh/install | bash
```

After install, Bun lives at `~/.bun/bin/bun`. Add it to your PATH if your shell doesn't pick it up automatically:

```bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

Add those two lines to your `~/.zshrc` or `~/.bashrc` to make them permanent.

### 2. Install the iMessage plugin

Inside a running Claude Code session:

```
/plugin install imessage@claude-plugins-official
```

If Claude Code reports the plugin is not found:

```
/plugin marketplace add anthropics/claude-plugins-official
```

Then retry the install. After installing:

```
/reload-plugins
```

**Grant Full Disk Access.** The iMessage plugin reads `~/Library/Messages/chat.db`, which macOS protects. When the plugin first runs, macOS prompts for access — click **Allow**. The prompt names whichever app launched Bun (Terminal, iTerm, your IDE). If the prompt does not appear or you clicked Don't Allow:

1. Open **System Settings > Privacy & Security > Full Disk Access**
2. Add your terminal application
3. Restart your terminal session

Without Full Disk Access, the plugin exits immediately with `authorization denied`.

### 3. Send your first message to Claude over iMessage

Restart Claude Code with the iMessage channel enabled:

```bash
claude --channels plugin:imessage@claude-plugins-official
```

Open Messages on any device signed into your Apple ID and send a message to **yourself**. Self-chat bypasses access control automatically — no pairing needed.

A few things to know:
- **Automation prompt:** The first time Claude replies, macOS asks if your terminal can control Messages. Click **OK**.
- **Duplicate notifications:** Messages may arrive twice in the session (two channel events per message). This is normal — Claude handles it once.
- **Echo behavior:** Messages Claude sends back echo as incoming channel events. Ignore them.
- **Allowing other contacts:** By default only your self-chat passes through. To allow another contact: `/imessage:access allow +1XXXXXXXXXX`

Your `chat_id` for self-chat will look like `iMessage;-;+1XXXXXXXXXX`.

### 4. Create Telegram bots (one per project)

Open [BotFather](https://t.me/BotFather) in Telegram and follow this flow for each project:

```
You: /newbot
BotFather: What's the display name?
You: My Research Assistant          ← human-readable, any text
BotFather: What's the username?
You: yourproject_bot                ← must end in "bot"; letters/digits/underscores only; 5-32 chars
BotFather: Done. Token: 1234567890:ABCDEFabcdef...
```

**Naming tips:**
- Username must end in `bot`
- Only letters, digits, and underscores
- Keep it short and opaque if you don't want the bot discoverable — anyone who knows a bot's username can message it
- Good pattern: `yourinitialsprojectname_bot` (e.g., `am_research_bot`)

**Store tokens securely.** Each token is a credential. Put it in `.claude/settings.local.json` (covered in step 6), which should be gitignored. Do not paste tokens into checked-in files or commit them.

### 5. Install the Telegram plugin

Inside a running Claude Code session:

```
/plugin install telegram@claude-plugins-official
```

If not found, first add the marketplace:

```
/plugin marketplace add anthropics/claude-plugins-official
```

Then retry. After installing:

```
/reload-plugins
```

### 6. Wire per-project settings.local.json

Each project's `.claude/settings.local.json` should configure only that project's Telegram bot token. This ensures each Claude Code session running in that directory picks up the right bot — not the wrong one.

Create or edit `.claude/settings.local.json` in each project directory:

```json
{
  "env": {
    "TELEGRAM_BOT_TOKEN": "1234567890:ABCDEFabcdef..."
  }
}
```

**Important:**
- Use `settings.local.json`, not `settings.json`. The `.local` variant is gitignored by convention and won't be committed.
- Do NOT put tokens in `~/.claude/settings.json` (global) — that would make every session try to use the same bot.
- The official docs also show a `/telegram:configure <token>` command which saves to `~/.claude/channels/telegram/.env`. For multi-bot setups, use the per-project env var approach instead, so each project's session uses its own token.

Verify your `.gitignore` (or global gitignore) includes `settings.local.json`:

```
.claude/settings.local.json
```

### 7. Set up persistent tmux sessions

Each project needs a tmux session that starts Claude with the Telegram channel enabled and stays running. Create one per project:

```bash
# Create a new detached tmux session for this project
tmux new-session -d -s project-name -c /path/to/project/directory

# Start Claude Code inside the pane with the Telegram channel
tmux send-keys -t project-name "claude --channels plugin:telegram@claude-plugins-official" Enter
```

Replace `project-name` with a short identifier (e.g., `research`, `twitter-bot`, `tech-support`) and `/path/to/project/directory` with the actual path.

To attach to a running session later:

```bash
tmux attach -t project-name
```

To detach without stopping it: `Ctrl-b` then `d`.

**Auto-starting sessions on boot** (optional): Add a launchd plist or a login item that runs `tmux new-session ...` on startup. The pattern is the same as RC automation but without `claude remote-control` — just `claude --channels plugin:telegram@claude-plugins-official` in the tmux pane.

You can pass multiple channel plugins to a single session, space-separated:

```bash
claude --channels "plugin:telegram@claude-plugins-official plugin:imessage@claude-plugins-official"
```

### 8. Pair your Telegram account

For each bot, complete the pairing flow. The session must be running with `--channels` active.

1. Open Telegram and send **any message** to your bot (e.g., "hello")
2. The bot replies with a 6-character pairing code
3. In your Claude Code session terminal (attach to the tmux pane), run:
   ```
   /telegram:access pair <code>
   ```
4. Lock down the allowlist so only your account can send messages:
   ```
   /telegram:access policy allowlist
   ```

After this, messages from your Telegram account route to Claude. Anyone else is silently dropped.

### 9. Verify

For each bot:
1. Send a test message from your phone: "what directory are you in?" or "summarize your project"
2. Confirm the reply comes back in Telegram
3. Confirm it's the right assistant (right project context)

If a bot doesn't respond, check: (a) is the tmux session running? (b) is Claude running with `--channels` inside it? (c) is the token correct in `settings.local.json`? (d) did you complete the pairing flow?

---

## Customization

**Adding more bots later:** Create a new BotFather bot, add its token to the new project's `settings.local.json`, create a tmux session, start Claude with `--channels`, and pair. No changes needed to existing sessions.

**Using Discord instead of Telegram:** The flow is nearly identical. Create a Discord bot in the [Discord Developer Portal](https://discord.com/developers/applications), enable Message Content Intent, invite it to your server, install the plugin with `/plugin install discord@claude-plugins-official`, configure with `/discord:configure <token>`, start with `claude --channels plugin:discord@claude-plugins-official`, pair via DM → `/discord:access pair <code>` → `/discord:access policy allowlist`.

**iMessage on a separate Apple ID:** The iMessage plugin reads `chat.db` on the local machine, tied to the Apple ID signed into the macOS Messages app. If you want a separate identity, sign a different Apple ID into a secondary user account.

**Multiple Macs:** Each Mac runs its own Claude Code sessions. You would create separate bots for each machine's sessions (BotFather bot tokens are per-bot, not per-device). The iMessage plugin works on whichever Mac has Messages signed in with your Apple ID.

**Bot username discretion:** Telegram bot usernames are public — anyone who guesses the username can message your bot (they'll be silently dropped by the allowlist, but they can try). Using opaque names reduces discoverability. Do not include your name, institution, or project names in the username if that matters to you.

**No Claude tokens, no relay cost:** Channel messages are delivered by the plugin polling the Telegram/Discord API or reading `chat.db` locally. There is no relay server between your phone and your Mac, and no per-message API cost beyond your normal Claude Code usage.

---

## Troubleshooting

**iMessage plugin doesn't see messages / exits immediately**
- Missing Full Disk Access. Open **System Settings > Privacy & Security > Full Disk Access** and add your terminal app.

**Telegram bot doesn't respond**
- Is the tmux session running? `tmux ls`
- Is Claude running inside it with `--channels`? Attach and check: `tmux attach -t project-name`
- Is the token correct? Check `.claude/settings.local.json` in the project directory.
- Did you complete the pairing flow? The bot only responds to paired accounts.

**Wrong project context — bot answers as the wrong assistant**
- Each tmux session must be started from the correct project directory. Check the `-c` flag in your `tmux new-session` command.
- Check that each project's `settings.local.json` has only that project's token.

**Permission prompts block responses while you're away**
- When Claude hits a permission prompt (tool approval), the session pauses until you respond.
- Options: (a) attach to the tmux pane and approve; (b) add the tool to your project's allowlist in `.claude/settings.local.json`; (c) reply "yes [code]" through the channel itself if the channel plugin supports permission relay.
- For fully unattended use, `--dangerously-skip-permissions` bypasses prompts entirely — only use this in environments you trust.

**Plugin not found on install**
- Run `/plugin marketplace add anthropics/claude-plugins-official` first, then retry.
- If still failing: `/plugin marketplace update claude-plugins-official`

**Duplicate messages handled twice**
- iMessage delivers two channel events per message (normal behavior). Claude Code handles this internally; if you see duplicate responses, add logic in your session's CLAUDE.md to ignore echoes, or [VERIFY: whether the official plugin has a dedup flag].

**Claude Code version too old**
- Channels require v2.1.80 or later. Run `claude --version` to check. Upgrade with however you installed Claude Code (typically `npm update -g @anthropic-ai/claude-code`).

**`console and API key authentication is not supported`**
- Channels require a claude.ai login, not an API key. Run `claude login` and authenticate with your claude.ai account.

---

## What about Remote Control?

RC is still useful when you need to **watch Claude work in real time** — streaming output, interrupting mid-task, guiding step-by-step. If you're doing a long interactive session and you need the full terminal experience on your phone, RC is the right tool.

Channels win for **task-based requests** on unreliable networks, **multi-project setups** where each assistant needs its own identity, and any workflow where local MCP servers need to work reliably.

Anthropic ships both for a reason. The comparison table from the official docs:

| Feature | Good for |
|---|---|
| Remote Control | Steering an in-progress session while away from your desk |
| Channels | Pushing events into your already-running local session from phone or external systems |

You can run both from the same Mac — just not from the same tmux session simultaneously. Keep RC sessions and channels sessions in separate panes.

---

## License

MIT. See [LICENSE](../../LICENSE) in the repo root.
