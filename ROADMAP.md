# Roadmap: curio-byte

## Goal & Architecture Principles

- **Primary Objective:** Deliver an all-in-one Discord showcase companion, AI assistant, automated disaster recovery backup daemon, and administrative moderation suite for the **Code, Build, Repeat** server.
- **Key Principles & Constraints:**
  - **Zero Data Loss & Disaster Recovery:** Regularly snapshot complete guild structures (channels, roles, members, webhooks, emojis, permissions) with atomic writes to guarantee fast offline disaster recovery.
  - **Provider Resilience:** High availability across AI interactions via multi-tier fallback routing across five distinct providers (Groq, Cloudflare Workers AI, Google Gemini, OpenRouter, Venice AI).
  - **Fast, Local Persistence:** High-performance, concurrent SQLite storage utilizing Write-Ahead Logging (`WAL`) mode via `better-sqlite3` for polls, user settings, guild configurations, and command analytics.
  - **Zero-Elevation & Privacy:** Run strictly in user space under systemd; protect private message text and sensitive credentials via strict `.private/` and git-ignored directories; mandatory cryptographic GPG commit signing.

---

## Phase 0 — Design & Core Client Foundations (COMPLETED)

- [x] Establish Node.js project architecture with Discord.js v14 and Gateway intents (`Guilds`, `GuildMembers`, `GuildMessages`, `MessageContent`, `GuildVoiceStates`, `GuildModeration`).
- [x] Implement multi-stage configuration loader reading credentials from `.private/config.json`.
- [x] Build automated REST v10 global slash command registration on client startup (`registerCommandsAutomatically`).
- [x] Implement foundational showcase commands:
  - `/about` — Bot identity and motto.
  - `/projects` — Featured portfolio showcase.
  - `/languages` — Supported technologies and tools.
  - `/invite` — Ephemeral invite link generation with configured permissions.
  - `/help` — Contextual slash command overview.

**Exit Criteria Met:** Operational Discord bot connected to the Gateway, capable of auto-registering global slash commands and responding to core showcase interactions.

---

## Phase 1 — Portfolio Search, AI Cascade & Moderation (COMPLETED)

- [x] Implement `/project [query]` with in-memory fuzzy search across portfolio project names, descriptions, and tags.
- [x] Build 5-tier AI prompt routing cascade (`src/ai-prompt-command.js`):
  - Tier 1: Groq (`openai/gpt-oss-20b`).
  - Tier 2: Cloudflare Workers AI (`@cf/zai-org/glm-4.7-flash`).
  - Tier 3: Google Gemini (`gemini-3.8-flash` with automatic fallback to `gemini-3.7-flash`).
  - Tier 4: OpenRouter (`openrouter/free`).
  - Tier 5: Venice AI (`llama-3.3-70b`).
- [x] Implement `/ai-prompt test` for isolated single-provider health checks.
- [x] Implement message splitting utility (`splitResponse`) to cleanly format outputs exceeding Discord's 2,000-character limit.
- [x] Build complete administrative moderation suite (`src/admin-commands.js`):
  - Member actions: `/moderation ban`, `/moderation kick`, `/moderation timeout`.
  - Warning system: `/moderation warn`, `/moderation warnings`, `/moderation clear-warnings` (persisted to `.private/warnings.jsonl`).
  - Message management: `/moderation purge` for bulk message deletion.
  - Channel management: `/channel create`, `/channel edit`, `/channel delete`, `/channel lock`, `/channel unlock` (lock state saved to `.private/channel-locks.json`).
  - Role management: `/role create`, `/role edit`, `/role delete`, `/role add`, `/role remove`, `/role permission`.
  - Server details: `/server info`.

**Exit Criteria Met:** Multi-provider AI query cascade operational with automatic fallbacks and a comprehensive administrative moderation suite.

---

## Phase 2 — Persistence, Automated Backups & Auditing (COMPLETED / CURRENT)

- [x] Implement SQLite persistence layer (`src/db.js`) using `better-sqlite3` in WAL mode:
  - `bot_analytics` — Command usage telemetry by user, guild, and timestamp.
  - `guild_settings` — Guild-specific log/chat channel IDs and audit status.
  - `user_settings` — Per-user themes and notification toggles.
  - `polls` & `poll_votes` — Interactive poll state and vote accounting.
- [x] Build interactive button polling (`/poll`):
  - Generates interactive Discord button rows.
  - Dynamically recalculates percentages and tallies in embed descriptions.
  - Prevents duplicate voting while allowing vote changes.
  - Closes cleanly after 5 minutes with disabled button states.
- [x] Implement server health & telemetry dashboard (`/analytics`):
  - Live member metrics (total, humans, bots).
  - Channel allocations (text, voice, categories).
  - Governance & role statistics.
  - Top SQLite command invocation breakdown.
  - Backup storage telemetry (snapshot counts, latest size, oldest snapshot).
- [x] Build automated 4-hour full-guild backup daemon (`src/server-backup.js`):
  - Collects channels, active threads, roles, members, webhooks, emojis, stickers, bans, invites, AutoMod rules, events, and welcome screens.
  - Atomic write and rename via `.tmp` file with `0o600` permissions.
  - Rolling retention policy maintaining 60 snapshots per guild.
- [x] Build private channel provisioning (`/curio-byte setup`):
  - Creates or secures private `Curio.Byte` category with `#logs` and `#chats`.
  - Attaches audit listeners for 16 distinct server event types.
- [x] Configure Linux systemd user daemon (`curio-byte.service`) with auto-restart supervision.
- [x] Comprehensive documentation overhaul:
  - [README.md](README.md) — System architecture, capabilities, setup, and deployment guides.
  - [AGENTS.md](AGENTS.md) — Mandatory GPG signing (`daniel@danielkill.online`), non-elevation, and test isolation.
  - [HANDOFF.md](HANDOFF.md) — Active operational status, configurations, and immediate next steps.
  - [ROADMAP.md](ROADMAP.md) — Multi-phase milestones and architecture roadmap.

**Exit Criteria Met:** Production bot actively running as a systemd user service with automated 4-hour backups, SQLite persistence, and full documentation sync.

---

## Phase 3 — Hardening, Test Isolation & Lifecycle Polish (PLANNED)

- [ ] **Test Runner Database Isolation:**
  - Ensure `test/server-setup.test.js` sets `process.env.DB_PATH = ':memory:'` before importing `src/server-setup.js` so `npm test` runs 100% cleanly without colliding with the live systemd daemon's SQLite write lock (`SQLITE_BUSY`).
- [ ] **Startup Stale Poll Reconciliation:**
  - Add a startup check in `src/index.js` to scan SQLite for active polls that were interrupted during a bot restart and cleanly mark them as closed (`is_closed = 1`).
- [ ] **Cold Backup Compression:**
  - Implement automatic gzip compression (`.json.gz`) for guild snapshots older than 7 days to reduce disk footprint in `server-backups/`.
- [ ] **Enhanced AI Pagination:**
  - Implement interactive pagination buttons or automatic private thread creation when an AI response spans across multiple chunks.

**Exit Criteria:** Flawless unit test execution during live daemon runs, automatic cold backup compression, and graceful poll recovery.

---

## Phase 4 — Disaster Recovery Restoration & Advanced Analytics (PLANNED - Future)

- [ ] **Guild Restoration CLI:**
  - Develop a standalone CLI script (`tools/restore-guild.js`) capable of reading a `server-backups/<guild_id>/latest.json` snapshot and selectively restoring missing channels, permission overwrites, or roles.
- [ ] **Time-Series Growth Analytics:**
  - Expand `/analytics` to display 7-day and 30-day member join/leave trends and command invocation trajectories from SQLite.
- [ ] **Webhook Alert Bridge:**
  - Optional external webhook dispatcher to notify server owners of critical moderation events or failed automated backup runs.
