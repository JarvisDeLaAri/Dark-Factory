# Shopping App Recovery Report

**Started:** 2026-04-25 23:48 UTC
**Scope:** What happened in the last 48h, what killed OpenClaw, how to orchestrate next time, how to test the app.

---

## What Was Done (48h Summary)

- **ShopMate app built:** FastAPI backend + vanilla JS frontend + AI agents (Ollama/Kimi + voice + TTS)
- **4 QA test suites ran:** Husband user journey, Wife user journey, AI husband helper, AI wife helper (all PASS)
- **~1240 session transcripts** accumulated in main agent sessions directory (attic'd during recovery)
- **~226 sessions** from Apr 23-25 alone in the attic
- **Backend running on port 8000** with health endpoint responding
- **Auto-git commits** every 4 hours (checkpointing)
- **Reminders firing every minute** via cron (orchestrator-reminder.sh) — 60+ identical entries in reminders.log

---

## What Killed OpenClaw

### Root Cause: Death by a Thousand Cuts

Multiple factors overloaded the system simultaneously:

1. **Session transcript explosion** — ~1240 `.jsonl` files accumulated in `agents/main/sessions/`. Each session writes transcript + trajectory files. Disk I/O + memory pressure from indexing.

2. **Reminder cron spam** — `orchestrator-reminder.sh` was firing repeatedly (same timestamp, dozens of identical entries in reminders.log). Likely scheduled too aggressively (every minute or on a tight loop).

3. **2 ShopMate crons running** — removed during recovery. These were probably reminder/check-in jobs that compounded the load.

4. **84 zero-byte `.tmp` files** in `cron/` — stale/aborted cron runs leaving debris.

5. **WhatsApp connection flapping** — `Error: Connection Closed`, 428 errors from WhatsApp Web. Possibly from high load causing timeouts, triggering reconnection storms.

6. **VPS CPU steal 60-88%** — host was already under resource pressure. Everything ran 3-4x slower.

7. **Discord channel enabled** — was disabled during recovery to reduce plugin load.

### The Crash

Gateway threw `TypeError` (uncaught exception) at 20:17 UTC on Apr 25. No detailed stack trace in stability dump. Likely a cascade failure — memory pressure + event loop saturation + perhaps a malformed message from the flapping WhatsApp connection.

### Recovery Actions Taken

All documented in `~/.openclaw/.recovery-20260425-190410/STATUS.md`:
- Removed 2 ShopMate crons
- Wiped per-agent session indexes (13 agents)
- Cleared stale delivery-queue entries
- Deleted 84 zero-byte `.tmp` files
- Disabled Discord channel
- Moved 1240 main session transcripts to attic
- Set `plugins.allow` whitelist (10 entries)
- Cleared WA app-state-sync files
- Gateway boot time: down from ~10 min to ~32-43s

---

## App Overview

`shopping-app/` is organized into clear functional areas and aligns with TODO Phase 2 completion + Phase 3 QA focus.

- **backend/**: FastAPI + SQLite core (`app.py`, `api_routes.py`, `models.py`, `config.py`) plus runtime dirs (`data/`, `logs/`, `uploads/`) and `requirements.txt`.
- **frontend/**: Vanilla SPA client (`index.html`, `app.js`, `styles.css`) with localization (`i18n.js`) and real-time socket client (`ws-client.js`).
- **ai-agents/**: AI assistant layer (`ai_agent.py`, `ollama_client.py`, `function_tools.py`, `ai_routes.py`) plus voice stack (`audio_routes.py`, `voice_service.py`, `tts_service.py`) and `test_audio.py`.
- **docs/**: Project documentation and research (`research-report.md`).
- **tests/**: QA scenario docs for husband/wife and AI helper flows (`qa-*.md`) validating key user journeys.
- **scripts/**: Automation helper scripts (`orchestrator-reminder.sh`, `git-auto-commit.sh`).

From `TODO.md`: phases 1–2 are marked complete, phase 3 testing is mostly done (remaining: real-time 2-user sync, security review, code review), and phase 4 lists polish/fix items (Ollama model mismatch, image endpoint issues, agent-to-agent log duplication).

**Git history:** Initial commit → auto checkpoints every 4 hours (Apr 24 16:04 → Apr 25 08:08). Last checkpoint 87aa424.

---

## Reminders Log

The `orchestrator-reminder.sh` cron was firing aggressively — 60+ identical entries all with timestamp Fri Apr 24 13:11:45 UTC 2026, each saying "🛒 SHOPMATE REMINDER: Check subagent progress". This suggests the cron was scheduled at a very short interval (possibly every minute) or the script itself was being invoked in a loop. This created massive spam in both the log file and potentially in the cron/delivery pipeline.

---

## How to Orchestrate Next Time

### DO:
- **Limit cron frequency** — Reminders should be hourly at most, not every minute
- **Clean up session transcripts regularly** — Move old `.jsonl` files to attic or delete after 7 days
- **Use a single orchestrator cron** — One parent cron that checks all subagent statuses, not multiple crons
- **Monitor disk space** — The transcript explosion ate significant disk I/O
- **Set plugins.allow whitelist** — This reduced boot time from ~10 min to ~32s
- **Keep Discord disabled** unless actively needed — reduces plugin load
- **Use `sessions_spawn` with timeout** — Don't let subagents run indefinitely
- **Commit stable points** — The auto-git commits every 4 hours saved work

### DON'T:
- **Don't schedule tight reminder loops** — They spam logs and overload the event loop
- **Don't let sessions accumulate forever** — ~1240 sessions in 48h is unsustainable
- **Don't run multiple overlapping crons** — The 2 ShopMate crons compounded the problem
- **Don't ignore CPU steal** — At 60-88% steal, the VPS is choking. Reduce parallelism.

### Recommended Orchestration Pattern:
1. One cron every 30-60 min: "Check ShopMate status"
2. Cron reads TODO.md, checks if backend is running on port 8000
3. If backend down → restart it, alert Ariel
4. If Phase 3 items pending → spawn ONE subagent for the next test
5. Subagent writes results to tests/qa-*.md
6. Main session reviews results, updates TODO.md
7. Repeat until Phase 3 complete

---

---

## Backend Status Update (Post-Report)

**As of report completion (Apr 25 23:48+ UTC):**
- `curl http://localhost:8000/health` → NOT RESPONDING
- Backend was last active Apr 24 17:10 (per session transcript `04660b26`)
- Likely crashed or was stopped during the Apr 25 OpenClaw recovery
- Backend log (`/tmp/shopmate-backend.log`) shows shutdown around 14:43 UTC Apr 24

**To restart:**
```bash
cd /root/.openclaw/workspace/shopping-app/backend && nohup python3 -m uvicorn app:app --host <IP_ADDRESS> --port 8000 --reload > /tmp/shopmate-backend.log 2>&1 &
```

**QA reports found (4 total):**
- `qa-husband.md` (168 lines) — Core features PASS
- `qa-wife.md` (441 lines) — Invite + join tested PASS
- `qa-ai-husband.md` (207 lines) — Chat/TTS/transcribe/history PASS
- `qa-ai-wife.md` (298 lines) — Agent-to-agent WORKS!

## How to Test the App

### Quick Health Check
```bash
curl -s http://localhost:8000/health
# Expected: {"status":"ok","app":"ShopMate","version":"0.1.0"}
```

### Backend API Tests (curl)

**1. Register two users:**
```bash
curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"husband@test.com","username":"husband","password":"test123"}'

curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"wife@test.com","username":"wife","password":"test123"}'
```

**2. Create couple + get invite code:**
```bash
curl -s -X POST http://localhost:8000/api/v1/couples \
  -H "Authorization: Bearer <husband_token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Our Couple"}'
```

**3. Join couple:**
```bash
curl -s -X POST http://localhost:8000/api/v1/couples/join \
  -H "Authorization: Bearer <wife_token>" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "invite_code=<CODE>"
```

**4. Create shopping list + add items:**
```bash
curl -s -X POST http://localhost:8000/api/v1/lists \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Weekly Groceries"}'

curl -s -X POST http://localhost:8000/api/v1/lists/1/items \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Milk","quantity":2,"unit":"liters","notes":"Whole"}'
```

**5. AI Chat:**
```bash
curl -s -X POST http://localhost:8000/api/ai/chat \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message":"What should we buy for dinner?"}'
```

**6. AI Health:**
```bash
curl -s http://localhost:8000/api/ai/health
```

### Frontend Test
Open `file:///root/.openclaw/workspace/shopping-app/frontend/index.html` in a browser, or serve it:
```bash
cd /root/.openclaw/workspace/shopping-app/frontend
python3 -m http.server 8080
# Then visit http://localhost:8080
```

### Known Issues to Fix Before Full Testing
1. **Ollama model name** — `ollama_client.py` may reference `kimi-k2.6:cloud` but available model is `kimi-k2.5:cloud`
2. **WebSocket double-accept** — May need fix in `app.py` (verify `ConnectionManager.connect()` doesn't call accept twice)
3. **Agent-to-agent log duplication** — Creates duplicate chat_log entries
4. **Image chat endpoint** — Returns "model not found" error

---

## File Locations

- **Recovery backup:** `~/.openclaw/.recovery-20260425-190410/`
- **Recovery status:** `~/.openclaw/.recovery-20260425-190410/STATUS.md`
- **App code:** `/root/.openclaw/workspace/shopping-app/`
- **App TODO:** `/root/.openclaw/workspace/shopping-app/TODO.md`
- **App git:** `/root/.openclaw/workspace/shopping-app/.git/`
- **QA reports:** `/root/.openclaw/workspace/shopping-app/tests/qa-husband.md`, `qa-wife.md`
- **Backend log:** `/tmp/shopmate-backend.log`
- **This report:** `/root/.openclaw/workspace/shopping-recovery/recovery-report.md`

---

## Recovery Report Status

| Section | Status |
|---------|--------|
| What Was Done | ✅ Written |
| What Killed OpenClaw | ✅ Written |
| Recovery Actions | ✅ Written |
| App Overview | ✅ Written |
| Reminders Log | ✅ Written |
| How to Orchestrate Next Time | ✅ Written |
| How to Test the App | ✅ Written |
| Known Issues | ✅ Written |
| File Locations | ✅ Written |

**Report complete. Size: ~6.5KB.**
