# Issue Catalog

> All issues discovered during initial HCI deployment on macOS ARM64 (Apple Silicon).
> Each issue links to the fix commit and affected source files.

| # | Issue | Severity | Fix Commit | Component |
|---|-------|----------|------------|-----------|
| 1 | `better-sqlite3` architecture mismatch | 🔴 Critical | `2f43cb5` | DB Layer |
| 2 | Session list — all message counts = 0 | 🔴 Critical | `2f43cb5` | Chat Page |
| 3 | Chat — clicking session shows no messages | 🔴 Critical | `2f43cb5` | Chat Page |
| 4 | Chat — sending message does nothing | 🔴 Critical | `2f43cb5` | Chat Page |
| 5 | Usage page — labels without values | 🔴 Critical | `2f43cb5` | Usage Page |
| 6 | Usage charts — blank / no data | 🔴 Critical | `2f43cb5` | Usage Page |
| 7 | Skills Hub — blank / "No skills found" | 🟡 Major | — | Skills Page |
| 8 | Session list — all "📝 Other", no filter | 🟡 Major | `2f43cb5` | Chat Sidebar |
| 9 | Cron sessions clutter chat list | 🟢 Minor | `2f43cb5` | Chat Sidebar |
| 10 | Agent name — model leaked into title | 🟢 Minor | `2f43cb5` | Agents Page |

---

## Issue 1: better-sqlite3 Architecture Mismatch

**Severity:** 🔴 Critical | **Component:** DB Layer  
**Affects:** All pages reading Hermes `state.db`  
**Fix commit:** `2f43cb5`

### Description

The `better-sqlite3` npm package ships a prebuilt native addon compiled for `x86_64`.
On Apple Silicon Macs (`arm64e`), every `new Database()` call throws:

```
dlopen(...better_sqlite3.node):
  incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

This is the **root cause of Issues 2–6**. All downstream fixes implement a
three-tier fallback: native → `sqlite3` CLI → `hermes` CLI.

### Files Changed

| File | Function/Route | Change |
|------|---------------|--------|
| `server.js:1384` | `loadSessionsFromDb()` | Added try/catch + `loadSessionsFromDbCli()` |
| `server.js:1400` | `loadSessionsFromDbCli()` | New function — `sqlite3 -json` query |
| `server.js:4068` | `GET /api/sessions/:id/messages` | Added CLI export fallback |
| `server.js:4204` | `GET /api/usage/:days` | Replaced with `getInsights()` delegation |
| `server.js:4284` | `GET /api/usage/daily/:days` | Added `getDailyViaCli()` fallback |
| `server.js:4400` | `getDailyViaCli()` | New function — 3 `sqlite3 -json` queries |

### Recovery

```bash
git am patches/better-sqlite3-compat.patch
```

---

## Issue 2: Session List — All Message Counts = 0

**Severity:** 🔴 Critical | **Component:** Chat Page (sidebar)  
**Depends on:** Issue 1 (better-sqlite3)  
**Fix commit:** `2f43cb5`

### Symptom

Every session in the chat sidebar shows "0 msgs" regardless of actual message count.

### Root Cause

`loadSessionsFromDb()` in `getAllSessions()` uses `new Database()` which crashes
(Issue 1). Without DB data, the `message_count` field is never populated, defaulting
to 0 in the frontend.

### Fix

Added `loadSessionsFromDbCli()` — executes `sqlite3 -json` to query `message_count`
directly. Also returns `source` field for filter categorization.

---

## Issue 3: Chat — Clicking Session Shows No Messages

**Severity:** 🔴 Critical | **Component:** Chat Page (message viewer)  
**Depends on:** Issue 1 (better-sqlite3)  
**Fix commit:** `2f43cb5`

### Symptom

Clicking any session in the sidebar selects it (📌) but the main area shows
"Welcome to Chat" instead of message history. No error visible to user.

### Root Cause

`GET /api/sessions/:id/messages` uses `new Database()` which crashes (Issue 1).
The catch block returns `{ok: false, error: ...}`, which the frontend doesn't
display — it just stays on the welcome screen.

Additionally, the initial fallback had a bug: `tool_calls` field could be either
a string (`JSON.parse` needed) or an already-parsed object. Doing `JSON.parse()`
on an object threw, and the silent catch swallowed ALL messages.

### Fix

1. Try native DB read first  
2. On fail → `execHermes(['sessions', 'export', ...], 60000)` with 60s timeout  
3. Parse JSONL: extract `messages` array from each exported session object  
4. **Critical:** Check `typeof toolCalls === 'string'` before `JSON.parse`

---

## Issue 4: Chat — Sending Message Does Nothing

**Severity:** 🔴 Critical | **Component:** Chat Page (input)  
**Fix commit:** `2f43cb5`

### Symptom

Typing a message and pressing Enter/Send shows no response. The user message
appears in the chat area but the assistant never responds. No error shown.

### Root Cause

The `sendChatMessage()` function tries WebSocket first. The WS connection is
unstable (frequent `[WS] ping timeout, forcing reconnect`). When `wsClient.connected`
is `true`, it calls `sendViaWebSocket()` which creates a Promise that **never
resolves or rejects** if the WS drops after the check. The CLI fallback code
(`sendViaCLI`) is unreachable.

### Fix

Added a 15-second timeout in `sendViaWebSocket()`:

```js
const wsTimeout = setTimeout(() => {
  wsClient.removeEventListener('message', onDone);
  reject(new Error('WebSocket timeout (15s)'));
}, 15000);
```

When timeout fires, Promise rejects → `catch(wsErr)` logs the warning →
falls through to `sendViaCLI()` which executes `hermes chat -Q`.

### Files Changed

| File | Function | Change |
|------|----------|--------|
| `src/js/main.js:1771` | `sendViaWebSocket()` | Added 15s timeout with clearTimeout on all paths |

---

## Issue 5: Usage Page — Labels Without Values

**Severity:** 🔴 Critical | **Component:** Usage Page  
**Depends on:** Issue 1 (better-sqlite3)  
**Fix commit:** `2f43cb5`

### Symptom

Usage page shows column headers ("Sessions", "Messages", "Input Tokens", etc.)
but all values are blank/zero.  
Agent filter dropdown only shows "All agents" (no individual profiles).

### Root Cause

`GET /api/usage/:days` contained 100+ lines of complex DB aggregation logic
with multiple `new Database()` calls. The first call crashed (Issue 1), and
the catch block returned `{ok: false, error: ...}`.

### Fix

Replaced the entire endpoint body with a single delegation to `getInsights()`:

```js
const data = await getInsights(days, profile);
res.json({ ok: true, ...data, period: ... });
```

`getInsights()` works because it calls `shell('hermes insights --days N')`,
which goes through `bash -lc` and finds `hermes` in the user's shell PATH.

---

## Issue 6: Usage Charts — Blank / No Data

**Severity:** 🔴 Critical | **Component:** Usage Page (charts)  
**Depends on:** Issue 1 (better-sqlite3)  
**Fix commit:** `2f43cb5`

### Symptom

"Daily Token Trend", "Daily Cost", and "Model Distribution" canvas areas are
blank. "Models", "Platforms" sections show "No data".

### Root Cause

`GET /api/usage/daily/:days` queries `state.db` via `new Database()` which
crashes (Issue 1).

### Fix

Added `getDailyViaCli()` — 3 separate `sqlite3 -json` queries for:

1. **Daily aggregation** — `SELECT DATE(started_at, ...) as date, COUNT(*), SUM(...) GROUP BY date`
2. **Hourly distribution** — `SELECT CAST(strftime('%H', ...) AS INTEGER), COUNT(*), SUM(...) GROUP BY hour`
3. **Totals** — `SELECT COUNT(*), SUM(input_tokens), SUM(output_tokens), SUM(message_count)`

Returns `byModel: []` and `byPlatform: []` (these are not available via CLI),
so the Models/Platforms lists show "No data" as a known limitation.

---

## Issue 7: Skills Hub — Blank / "No skills found"

**Severity:** 🟡 Major | **Component:** Skills Page

### Symptom

Skills Hub shows the page header and search bar, but the skill list area says
"No skills found on page 1". Clicking Refresh doesn't help.

### Root Cause

`execHermes(['skills', 'browse', '--page', '1'])` calls `execFile('hermes', ...)`
directly (no bash wrapper). launchd starts HCI with a limited PATH:
```
/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin
```
`hermes` lives at `~/.local/bin/hermes` which is NOT in launchd's PATH.

The `shell()` function works (it uses `bash -lc`) but `execHermes()` does not.

### Fix

Added `~/.local/bin` to the launchd plist:

```xml
<key>PATH</key>
<string>/Users/chengjiahui/.local/bin:/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
```

### Files Changed

| File | Change |
|------|--------|
| `~/Library/LaunchAgents/com.hermes.control-interface.plist` | Added PATH entry |

---

## Issue 8: Session List — All "📝 Other" / No Filter Options

**Severity:** 🟡 Major | **Component:** Chat Sidebar  
**Fix commit:** `2f43cb5`

### Symptom

The source filter dropdown only shows "All" and "📝 Other" — no way to filter
by CLI, Weixin, or cron sessions.

### Root Cause

`normalizeSource(s)` in `src/js/main.js` mapped `null` source to `'other'`.
CLI-parsed sessions have no `source` field (the CLI table output doesn't include
a source column), so every session got `_source = 'other'`.

### Fix

Changed `normalizeSource()`:
```js
// Before
if (!src) return 'other';
// After
if (!src) return 'cli';
```

Also added `COALESCE(source, 'cli')` in the `loadSessionsFromDbCli()` SQL query.

---

## Issue 9: Cron Sessions Clutter Chat List

**Severity:** 🟢 Minor | **Component:** Chat Sidebar  
**Fix commit:** `2f43cb5`

### Symptom

Hundreds of cron job sessions (with titles like "cron_...") fill the chat
sidebar, pushing user-initiated conversations far down the list or off-screen.

### Root Cause

The default SQL query fetched ALL sessions, including cron jobs. Since cron
jobs run every few minutes, they have recent `ended_at` timestamps and sort
to the top.

### Fix

Added `WHERE source IS NULL OR source != 'cron'` in `loadSessionsFromDbCli()`.

---

## Issue 10: Agent Name — Model Leaked Into Profile Name

**Severity:** 🟢 Minor | **Component:** Agents Page

### Symptom

On the Agents page, the profile `product-manager` with model `ark-code-latest`
appears as:
```
product-manager ark-code-latest Status ○ Stopped Model stopped
```
The model name is merged into the profile name field.

### Root Cause

The HCI table parser uses `split(/\s{2,}/)` to separate columns. The profile
name `product-manager` (15 chars) and model `ark-code-latest` (15 chars) are
separated by only 1 space because the profile nearly fills its column width.
The regex requires 2+ spaces, so the split merges them.

### Fix

Renamed profile from `product-manager` to `product` (7 chars), ensuring at
least 8 spaces before the model name starts.

### Files Changed

None (CLI rename only):
```bash
hermes profile rename product-manager product
```
