# HCI Deployment Fixes & Engineering Notes

> Hermes Control Interface — local deployment on macOS ARM64 (Apple Silicon)

---

## ⚠️ Core Issue: better-sqlite3 Architecture Mismatch

**Problem:** The `better-sqlite3` native binary was compiled for `x86_64` but this Mac uses `arm64e`.  
Every `new Database()` call throws:
```
dlopen(...better_sqlite3.node): incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

**Affects:** Every page/endpoint that reads `state.db` directly.

**Global fix strategy:** Three-tier fallback:
1. Try `new Database()` (native addon, works on compatible arch)
2. On fail → use `sqlite3` CLI (`execFileSync('sqlite3', ['-json', dbPath, sql])`)
3. On fail → use `shell('hermes <subcommand>')` CLI (via `bash -lc`, always works)

---

## Individual Fixes

### 1. Session List — Message Count = 0

**File:** `server.js` — `loadSessionsFromDb` function

**Before:** `new Database()` crash → `dbSessions` empty → message count = 0 for all sessions.

**After:** `loadSessionsFromDbCli()` fallback:
```js
execFileSync('sqlite3', ['-json', stateDbPath,
  `SELECT id, COALESCE(message_count, 0) as message_count, COALESCE(source, 'cli') as source
   FROM sessions WHERE source IS NULL OR source != 'cron'
   ORDER BY COALESCE(ended_at, started_at) DESC, id DESC LIMIT ${limit}`
]);
```
- Also adds `source` column for filter dropdown
- Excludes cron sessions from main list

### 2. Chat Messages — Clicking Session Hangs / Shows 1 Message

**File:** `server.js` — `/api/sessions/:id/messages` endpoint

**Before:** `new Database()` crash → fallback didn't exist → frontend showed error.

**After:** Two-tier fallback:
1. Try native DB read
2. On fail → `execHermes(['sessions', 'export', tmpFile, '--session-id', sessionId], 60000)`
   - Read exported JSONL, extract messages array
   - Handle `tool_calls` as both string (needs JSON.parse) and object (already parsed)
   - Fix: `typeof toolCalls === 'string' ? JSON.parse(toolCalls) : toolCalls`

### 3. Send Message — No Response

**File:** `src/js/main.js` — `sendViaWebSocket` function

**Before:** WebSocket connection flaky (keeps timing out). `sendViaWebSocket` Promise never resolves or rejects → CLI fallback never runs → user sees no response.

**After:** Added 15-second timeout:
```js
const wsTimeout = setTimeout(() => {
  wsClient.removeEventListener('message', onDone);
  reject(new Error('WebSocket timeout (15s)'));
}, 15000);
```
When WS times out, Promise rejects → CLI fallback kicks in.

### 4. Skills Hub — Blank Page

**File:** launchd plist `com.hermes.control-interface.plist`

**Before:** `execHermes('skills browse --page 1')` failed because `hermes` wasn't in launchd PATH.
```
PATH=/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin   ← missing ~/.local/bin
```

**After:**
```xml
<key>PATH</key>
<string>/Users/chengjiahui/.local/bin:/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
```

### 5. Agent List Display — Name/Model Merged

**File:** (None — renamed profile instead of modifying HCI parser)

**Before:** `product-manager` (15 chars) + `ark-code-latest` (15 chars) → only 1 space between → `split(/\s{2,}/)` parser merged them.

**After:** Renamed profile to `product` (7 chars). HCI parser unchanged.

### 6. Usage Page — No Data / "Models No data" / "Platforms No data"

**File:** `server.js` — `/api/usage/:days` endpoint and `/api/usage/daily/:days` endpoint

**Before:** Both endpoints used `new Database()` → crashed → `{ok: false, error: ...}` → frontend showed labels without values.

**After:**
- `/api/usage/:days` → delegates to `getInsights()` which uses `shell('hermes insights --days N')` (always works, goes through bash)
- `/api/usage/daily/:days` → try native DB, fallback to `getDailyViaCli()` which uses:
  ```js
  execFileSync('sqlite3', ['-json', stateDbPath, `SELECT ... GROUP BY date ORDER BY date`]);
  ```
  Returns daily, byHour, and totals.

### 7. Session Source Label — All "📝 Other"

**File:** `src/js/main.js` — `normalizeSource` function

**Before:** `if (!src) return 'other'` → CLI sessions (no source field) all lumped into "Other".

**After:** `if (!src) return 'cli'` → CLI sessions show as "⌨️ CLI".

### 8. Cron Sessions Cluttering Chat List

**File:** `server.js` — `loadSessionsFromDbCli` SQL query

**Before:** `SELECT ... FROM sessions ORDER BY ...` — returned ALL sessions including cron jobs.

**After:** Added `WHERE source IS NULL OR source != 'cron'` to exclude cron.

### 9. Home Page — "Loading..." for Agents/Gateways

This is a transient issue caused by WebSocket disconnects. Not a code bug.

---

## Engineering Practices

### Git Workflow

```bash
# Check current state
cd ~/hermes-control-interface
git status

# Stage and commit fixes separately per concern:
git add server.js
git commit -m "fix: better-sqlite3 arch mismatch — add sqlite3 CLI fallback everywhere

- loadSessionsFromDb: secondary sqlite3 CLI path
- /api/sessions/:id/messages: export CLI fallback (60s timeout)
- /api/usage/:days: delegate to hermes insights
- /api/usage/daily/:days: sqlite3 CLI query for daily aggregates
- loadSessionsFromDbCli: exclude cron sessions, add source field"

git add src/js/main.js
git commit -m "fix: Chat send hangs on WS timeout + session source labels

- sendViaWebSocket: add 15s timeout so CLI fallback runs
- normalizeSource: null → 'cli' instead of 'other'"

git add dist/ dist/index.html
git commit -m "chore: rebuild dist after frontend fixes"
```

### Branch Strategy

```bash
# Create a fixes branch for maintainability:
git checkout -b fix/better-sqlite3-compat
# ... commit fixes ...
git checkout main
git merge fix/better-sqlite3-compat
```

### Upstream Update Handling

When upstream releases a new version:

```bash
# Save current fixes to a patch file
git format-patch origin/main..HEAD --stdout > ~/hci-fixes.patch

# Pull upstream
git fetch origin
git checkout main
git pull origin main

# Re-apply fixes (may have conflicts — resolve manually)
git am ~/hci-fixes.patch

# If conflicts: fix them, then
git am --continue

# Rebuild frontend
npm run build

# Restart HCI
launchctl stop com.hermes.control-interface
launchctl start com.hermes.control-interface
```

### Backup the Patch File

```bash
# The patch is self-contained and can be applied to any HCI version:
git format-patch origin/main..HEAD --stdout > ~/hermes-control-interface/patches/better-sqlite3-compat.patch
```

---

## Architecture Overview

### better-sqlite3 — All Call Sites

| Location | Function/Endpoint | Native Addon | Fallback |
|---|---|---|---|
| `server.js:1387` | `loadSessionsFromDb()` | `new Database()` | `loadSessionsFromDbCli()` via sqlite3 |
| `server.js:4083` | `/api/sessions/:id/messages` | `new Database()` | `hermes sessions export` CLI |
| `server.js:4204` | `/api/usage/:days` | `new Database()` | `getInsights()` via `hermes insights` |
| `server.js:4283` | `/api/usage/daily/:days` | `new Database()` | `getDailyViaCli()` via sqlite3 |

### PATH Dependency

| Binary | Location | Needed By |
|---|---|---|
| `hermes` | `~/.local/bin/hermes` | `execHermes()`, `shell()`, launchd |
| `sqlite3` | `/usr/bin/sqlite3` | `loadSessionsFromDbCli()`, `getDailyViaCli()` |

The launchd plist must include `~/.local/bin` in PATH for `execHermes` (which uses `execFile` without bash).

### Frontend Build

HCI uses Vite. Source is `src/js/main.js`, build output is `dist/assets/index-*.js`.  
Always run `npm run build` after editing `src/js/main.js`.

---

## Quick Recovery After Upstream Update

```bash
# 1. Pull upstream
git pull origin main

# 2. Re-apply fixes patch
git am ~/hermes-control-interface/patches/better-sqlite3-compat.patch
# or manually if conflicts

# 3. Restore launchd PATH fix
# Edit ~/Library/LaunchAgents/com.hermes.control-interface.plist

# 4. Rebuild + restart
npm run build
launchctl unload ~/Library/LaunchAgents/com.hermes.control-interface.plist
launchctl load ~/Library/LaunchAgents/com.hermes.control-interface.plist
```
