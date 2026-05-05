# better-sqlite3 Compatibility on Apple Silicon

> Engineering documentation for the `better-sqlite3` architecture mismatch fix.
> Last updated: 2026-05-05 | Applies to: v3.5.1+

---

## Problem Statement

The `better-sqlite3` npm package ships a prebuilt native addon
(`build/Release/better_sqlite3.node`) compiled for `x86_64`. On Apple Silicon
Macs (`arm64e`), every call to `new Database(path)` throws:

```
dlopen(...better_sqlite3.node):
  incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

This breaks every HCI feature that reads the Hermes `state.db` SQLite database
directly.

---

## Architecture

### Data Flow

```
┌──────────────────────────────────────────────────────────┐
│                   HCI (Node.js)                          │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐   ┌─────────────┐  │
│  │ Native Path  │    │ sqlite3 CLI  │   │ hermes CLI  │  │
│  │ (try first)  │───→│ (fallback 1) │──→│ (fallback 2)│  │
│  │              │    │              │   │             │  │
│  │ new Database │    │ execFileSync │   │ execFile /  │  │
│  │ (better-     │    │ ('sqlite3',  │   │ shell()     │  │
│  │  sqlite3)    │    │  [...])      │   │ ('hermes')  │  │
│  └──────┬───────┘    └──────┬───────┘   └──────┬──────┘  │
│         │                  │                   │         │
│         │           ┌──────┴───────┐           │         │
│         │           │              │           │         │
│         │      ┌────▼────┐   ┌────▼────┐      │         │
│         └──────┤ state.db├───┤hermes   │      │         │
│                │ (SQLite) │   │ CLI     │      │         │
│                └─────────┘   └─────────┘      │         │
└──────────────────────────────────────────────────────────┘
```

### Call Site Summary

| # | File | Function/Route | Native | Fallback 1 | Fallback 2 |
|---|------|---------------|--------|------------|------------|
| 1 | `server.js:1384` | `loadSessionsFromDb()` | `new Database()` | `loadSessionsFromDbCli()` via `sqlite3 -json` | — |
| 2 | `server.js:4068` | `GET /api/sessions/:id/messages` | `new Database()` | `hermes sessions export` (60s timeout) | — |
| 3 | `server.js:4204` | `GET /api/usage/:days` | `new Database()` | `getInsights()` via `hermes insights` | — |
| 4 | `server.js:4284` | `GET /api/usage/daily/:days` | `new Database()` | `getDailyViaCli()` via `sqlite3 -json` | — |

---

## Fix Details

### 1. Session List — Message Count & Source

**File:** `server.js` — `loadSessionsFromDb()`

**Before:** Native Database() crash → empty dbSessions → all sessions show 0 messages.

**After:** Wrapped in try/catch + `loadSessionsFromDbCli()`:

```js
function loadSessionsFromDbCli(stateDbPath, limit = 250) {
  const raw = execFileSync('sqlite3', ['-json', stateDbPath,
    `SELECT id, COALESCE(message_count, 0) as message_count,
            COALESCE(source, 'cli') as source
     FROM sessions WHERE source IS NULL OR source != 'cron'
     ORDER BY COALESCE(ended_at, started_at) DESC, id DESC
     LIMIT ${parseInt(limit)}`
  ], { encoding: 'utf8', timeout: 5000 });
  const dbSessions = JSON.parse(raw);
  return { dbSessions, previewBySessionId: {}, lastActivityBySessionId: {} };
}
```

Key improvements:
- Uses `COALESCE(source, 'cli')` so CLI sessions get a proper source label
- Excludes cron sessions (`WHERE source IS NULL OR source != 'cron'`)
- Fallback only returns message_count + source (no preview/lastActivity)

### 2. Session Messages — Click to View

**File:** `server.js` — `GET /api/sessions/:id/messages`

**Before:** `new Database()` inside try → caught → returned `{ok: false}` → frontend showed error.

**After:** Two-tier try with CLI fallback:

1. Try native DB read (fast, full metadata)
2. On fail → `execHermes(['sessions', 'export', ...], 60000)`:
   - 60-second timeout (large sessions can have 800+ messages, 1.3MB JSONL)
   - Parse exported JSONL into `{id, role, content, tool_calls, timestamp}` array
   - **Critical fix:** `tool_calls` may be a string (needs `JSON.parse`) or already an object — check `typeof` before parsing

```js
let toolCalls = m.tool_calls || null;
if (typeof toolCalls === 'string') {
  try { toolCalls = JSON.parse(toolCalls); } catch { toolCalls = null; }
}
```

### 3. Usage Overview — Stats

**File:** `server.js` — `GET /api/usage/:days`

**Before:** 100+ lines of complex DB aggregation with multiple `new Database()` calls
across all profiles. First call crashed the entire endpoint.

**After:** Replaced entire body with delegation to existing `getInsights()`:

```js
const data = await getInsights(days, profile);
res.json({ ok: true, ...data, period: ... });
```

`getInsights()` already works — it calls `shell('hermes insights --days N')` which
goes through `bash -lc` and always finds `hermes` in the user's shell PATH.

### 4. Usage Charts — Daily Breakdown

**File:** `server.js` — `GET /api/usage/daily/:days`

**Before:** Same `new Database()` crash → empty response.

**After:** Native try → `getDailyViaCli()` fallback:

Three separate `sqlite3 -json` queries for:
- **Daily aggregation:** `SELECT DATE(started_at, ...) as date, COUNT(*), SUM(input_tokens), SUM(output_tokens) ... GROUP BY date`
- **Hourly distribution:** `SELECT CAST(strftime('%H', ...) AS INTEGER) as hour, COUNT(*), SUM(...) ... GROUP BY hour`
- **Totals:** `SELECT COUNT(*), SUM(input_tokens), SUM(output_tokens) ...`

### 5. Send Message — WS Timeout

**File:** `src/js/main.js` — `sendViaWebSocket()`

**Before:** Promise never resolves on WS disconnect → CLI fallback never runs.

**After:** Added 15-second timeout:

```js
const wsTimeout = setTimeout(() => {
  wsClient.removeEventListener('message', onDone);
  reject(new Error('WebSocket timeout (15s)'));
}, 15000);
```

Cleared on success/error/not-connected.

---

## PATH Requirements

The following binaries must be in the launchd PATH for HCI to function:

| Binary | Default location | Used by |
|--------|-----------------|---------|
| `hermes` | `~/.local/bin/hermes` | `execHermes()`, `shell()` |
| `sqlite3` | `/usr/bin/sqlite3` | `loadSessionsFromDbCli()`, `getDailyViaCli()` |

**launchd plist fix (`com.hermes.control-interface.plist`):**

```xml
<key>PATH</key>
<string>/Users/chengjiahui/.local/bin:/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
```

---

## Impact Analysis

| Metric | Before (native only) | After (with fallbacks) |
|--------|---------------------|------------------------|
| Session list load | Crash → empty | Full list with message counts |
| Chat messages | Crash → error | Full message history (via export) |
| Usage overview | Crash → blank | Full stats (via hermes insights) |
| Daily charts | Crash → blank | Daily + hourly aggregations |
| Send message | WS hang → no response | WS timeout + CLI fallback |

---

## Recovery After Upstream Update

```bash
# 1. Pull upstream changes
git pull origin main

# 2. Re-apply compatibility patch
git am patches/better-sqlite3-compat.patch
# If conflicts: fix manually, then git am --continue

# 3. Restore launchd PATH
# Edit ~/Library/LaunchAgents/com.hermes.control-interface.plist

# 4. Rebuild frontend + restart
npm run build
launchctl stop com.hermes.control-interface
launchctl start com.hermes.control-interface
```

If the patch doesn't apply cleanly, use the interactive method:
```bash
# Save current fixes as a branch, then cherry-pick
git checkout -b fix/better-sqlite3-compat
git rebase main
# Resolve conflicts, then:
git checkout main
git merge fix/better-sqlite3-compat
```
