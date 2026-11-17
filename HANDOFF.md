# Active Task Handoff: curio-byte

## Current Operational State

Curio.Byte is deployed and actively running as a **systemd user service** (`curio-byte.service`) on host `danielpredpc` (WSL2 Linux environment).

- **Bot Identity:** `Curio.Byte#3577`
- **Runtime Environment:** Node.js v24.22.1 (`/home/daniel/.local/bin/node`) under user `daniel`
- **Systemd Unit:** `/home/daniel/.config/systemd/user/curio-byte.service` (Active and enabled)
- **Active Slash Commands:** 14 global slash commands auto-registered on startup
- **Database Engine:** SQLite 3 in WAL mode (`better-sqlite3`) located at `data/curio.db`
- **Automated Backup Engine:** Active 4-hour backup interval (`INTERVAL_MS = 14400000`) continuously snapshotting guild `1557930395868209162` to `server-backups/1557930395868209162/`
- **Documentation State:** Fully aligned with live service architecture ([README.md](README.md), [AGENTS.md](AGENTS.md), [ROADMAP.md](ROADMAP.md))

---

## Architectural & Subsystem Summary

| Subsystem | File / Location | Implementation Details |
|---|---|---|
| **Interaction Dispatcher** | `src/index.js` | Discord.js v14 client; auto-registers global slash commands via REST v10 on startup; routes chat commands and button interactions; captures command analytics into SQLite. |
| **Persistence Layer** | `src/db.js` | `better-sqlite3` with `PRAGMA journal_mode = WAL`. Manages tables: `bot_analytics`, `guild_settings`, `user_settings`, `polls`, and `poll_votes`. |
| **Interactive Polls** | `src/commands/poll.js` | Generates button components (`ActionRowBuilder`), records unique votes per user in SQLite, updates embed tallies and percentages dynamically, and closes automatically after 5 minutes. |
| **Server Analytics** | `src/commands/analytics.js` | Aggregates live guild metrics (humans, bots, channel types, roles), top SQLite command analytics, and snapshot storage telemetry (`latest.json` size, total backups). |
| **Portfolio Showcase** | `src/commands/project.js` | In-memory searchable index of Code, Build, Repeat portfolio projects (`/project [query]`, `/projects`, `/languages`). |
| **AI Routing Cascade** | `src/ai-prompt-command.js` | 5-tier fallback cascade: Groq (`gpt-oss-20b`) -> Cloudflare Workers AI (`glm-4.7-flash`) -> Google Gemini (`gemini-3.8-flash` / `gemini-3.7-flash`) -> OpenRouter (`openrouter/free`) -> Venice AI (`llama-3.3-70b`). Automatic message chunking for 2,000-character Discord limit. |
| **Backup Daemon** | `src/server-backup.js` | Runs every 4 hours (`4 * 60 * 60 * 1000` ms). Collects complete guild structure (channels, threads, roles, members, webhooks, emojis, stickers, bans, invites, AutoMod rules, events). Atomic write via `.tmp` file and rolling retention of 60 snapshots. |
| **Audit Logging Engine** | `src/server-setup.js` | `/curio-byte setup` provisions private `Curio.Byte` category with `#logs` and `#chats`. Event listener hooks into 16 server event types and formats color-coded embeds. |
| **Administrative Suite** | `src/admin-commands.js` | Moderation (/ban, /kick, /timeout, /purge), warning management (persisted to `.private/warnings.jsonl`), channel locks (persisted to `.private/channel-locks.json`), and granular role permission management. |

---

## Completed Milestones

- [x] Initialized Git repository, standard `.gitignore`, and protected `.private/` directory.
- [x] Built core Discord.js v14 client with Gateway intents for guilds, members, messages, voice, and moderation.
- [x] Implemented automatic startup registration for global slash commands.
- [x] Implemented SQLite persistence layer (`src/db.js`) with WAL mode for analytics, settings, and polls.
- [x] Developed `/poll` with interactive button voting, live percentage updates, and duplicate vote prevention.
- [x] Developed `/analytics` combining live Discord guild stats, SQLite command metrics, and backup storage analytics.
- [x] Built multi-tier AI prompt routing cascade (`/ai-prompt ask` and `/ai-prompt test`) supporting 5 providers with fallback logic.
- [x] Implemented automated 4-hour disaster recovery backup daemon with atomic JSON snapshots and rolling retention.
- [x] Built `/curio-byte setup` provisioning private channels and attaching 16-event audit logging.
- [x] Implemented full administrative moderation suite with warnings and channel locks.
- [x] Deployed and enabled systemd user service (`curio-byte.service`).
- [x] Comprehensive documentation overhaul: [README.md](README.md), [AGENTS.md](AGENTS.md), [HANDOFF.md](HANDOFF.md), and [ROADMAP.md](ROADMAP.md).

---

## In Progress & Immediate Next Steps

1. **Test Suite Database Isolation:**
   - In `test/server-setup.test.js`, set `process.env.DB_PATH = ':memory:'` before importing `src/server-setup.js`. Currently, running tests while the live systemd daemon is active can cause `SqliteError: database is locked` (`SQLITE_BUSY`) because `server-setup.js` defaults to `data/curio.db`.
2. **Backup Diff & Restoration Utility:**
   - Develop a CLI utility (`tools/restore-backup.js` or `tools/diff-backup.js`) to compare two snapshot JSON files or selectively restore missing channels or roles after an accidental deletion.
3. **AI Response Pagination & Threading:**
   - Expand `splitResponse()` in `src/ai-prompt-command.js` to optionally create a private thread or use pagination buttons when an AI response spans multiple Discord messages.
4. **Stale Poll Sweep on Startup:**
   - Add a startup check in `src/index.js` to inspect `polls` in SQLite and mark any open polls whose collector window elapsed while the bot was offline as `is_closed = 1`.

---

## Blockers & Gotchas

- **SQLite Database Lock During Tests (`SQLITE_BUSY`):**
  - Because `curio-byte.service` runs continuously in the background holding a write lock on `data/curio.db`, any test that imports `src/db.js` without setting `process.env.DB_PATH` to `:memory:` or a temporary file will fail with `SQLITE_BUSY`.
- **Privileged Gateway Intents:**
  - The bot requires **Server Members Intent** and **Message Content Intent** enabled in the Discord Developer Portal under Bot settings.
- **Git Commit Signing:**
  - Execute `git commit -S` inside WSL2 (`wsl.exe -d daniel-kill -e bash -c ...`) to utilize the active GPG agent and avoid Windows GUI pinentry prompt timeouts.
- **Remote Push Confirmation:**
  - Never push commits to remote repositories without explicit user instruction.
