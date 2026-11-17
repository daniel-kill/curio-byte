# Repository Agent Guidelines: curio-byte

Read [HANDOFF.md](HANDOFF.md) first, then [ROADMAP.md](ROADMAP.md), before starting work.
Before finishing any turn or session, record completed work, immediate next steps, and any blockers or gotchas in [HANDOFF.md](HANDOFF.md).

---

## Mandatory Commit Signing

Every commit created in this repository must have a valid cryptographic GPG signature:
- **Author / Signer:** Daniel Kill (`daniel@danielkill.online`)
- **Key Fingerprint:** `E453F9AEC380D33E4BA7FAC69B542AA270F867E5`
- **Signing Execution:** On `DANIELPREDPC`, execute signed git commits inside WSL2 (`daniel-kill` distro) to utilize the pre-authenticated `gpg-agent` session and avoid Windows GUI `pinentry.exe` desktop prompt timeouts:
  ```bash
  wsl.exe -d daniel-kill -e bash -c "cd /home/daniel/Repos/curio-byte && git commit -S -m 'type: description'"
  ```
- **Verification:** Always verify newly created commits:
  ```bash
  git log --show-signature -1
  ```
- **Remote Push Policy:** Never push commits to remote repositories (`git push`) without explicit confirmation from the user.

---

## Zero-Elevation & Security Policies

1. **Non-Elevated Execution:** Do not run commands requiring administrative or UAC elevation on Windows. On Linux/WSL, standard development and service commands run strictly under user mode (`systemctl --user`).
2. **Secrets & Privacy Isolation:**
   - Never commit API tokens, bot secrets, or configuration files containing credentials.
   - Local secrets reside strictly in `.private/` (`.private/config.json`, `warnings.jsonl`, `channel-locks.json`), which are protected by `.gitignore`.
   - Never commit live database files (`data/curio.db`) or guild backups (`server-backups/`). These files contain sensitive guild governance data, member information, webhook URLs with execution tokens, and audit trails.

---

## Architectural & Codebase Guardrails

1. **SQLite WAL Mode & Prepared Statements:**
   - Database operations in `src/db.js` use `better-sqlite3` with Write-Ahead Logging (`PRAGMA journal_mode = WAL`).
   - All SQL queries must be registered as prepared statements at module initialization to prevent SQL injection and guarantee performance.
2. **Test Environment Database Isolation:**
   - The production bot runs continuously as a background systemd user service, holding a write lock on `data/curio.db`.
   - When running unit tests (`npm test` / `node --test`), tests MUST configure `process.env.DB_PATH` to an isolated test file (or `:memory:`) before importing any application module that touches `src/db.js` or `src/server-setup.js` to prevent `SQLITE_BUSY` errors.
3. **Discord.js v14 Interaction Safety:**
   - Administrative and moderation commands must always respond ephemerally (`ephemeral: true`).
   - Always call `await interaction.deferReply({ ephemeral: true })` before performing operations that may exceed 3 seconds (such as AI provider requests, server provisioning, or large backup inspections).
   - Component collectors for buttons (e.g. in `/poll`) must cleanly handle collector timeouts and disable UI buttons upon expiration.
4. **Documentation Links:**
   - Always use relative markdown links between repository documents (e.g., `[README.md](README.md)`).

---

## Central Documentation & Knowledge Base

Consult central shared guides, runtimes, and agent documentation here:
- **Windows:** `C:\Users\danie\Repos\shared\docs\`
- **WSL:** `/home/daniel/Repos/shared/docs/`

Authoritative machine snapshots and installed packages:
- **Windows:** `C:\Users\danie\Repos\shared\data\snapshots\danielpredpc\`
- **WSL:** `/home/daniel/Repos/shared/data\snapshots\danielpredpc-wsl/`
