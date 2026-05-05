# Changelog

All notable changes to Hermes Control Interface will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [3.5.1] — 2026-05-05

### Fixed

#### better-sqlite3 Architecture Compatibility (Apple Silicon)

**Problem:** The `better-sqlite3` native addon was compiled for `x86_64` but the host
machine is `arm64e` (Apple Silicon Mac). Every call to `new Database()` threw:
```
dlopen(...better_sqlite3.node): incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

**Solution:** Three-tier fallback — try native addon, fall back to `sqlite3` CLI,
fall back to `hermes` CLI. Applied across all 4 database access points:

| Endpoint | Issue | Fix |
|---|---|---|
| Session list — messages always `0` | `loadSessionsFromDb` crashed | Added `loadSessionsFromDbCli()` via `sqlite3 -json` |
| Chat — click session hung/no messages | `/api/sessions/:id/messages` crashed | Added `hermes sessions export` JSONL fallback |
| Usage — labels without values | `/api/usage/:days` crashed | Delegated to `getInsights()` (`hermes insights` CLI) |
| Usage charts — blank | `/api/usage/daily/:days` crashed | Added `getDailyViaCli()` via `sqlite3 -json` |

Full documentation: `docs/BETTER_SQLITE3.md` | Replayable patch: `patches/better-sqlite3-compat.patch`

#### Chat Send — No Response

The WebSocket connection is unstable on this deployment (frequent ping timeouts).
`snedViaWebSocket()` created a Promise that never resolved or rejected when the WS
dropped, preventing the CLI fallback from running.

**Fix:** Added a 15-second timeout to the WebSocket send path. When hit, the Promise
rejects and execution falls through to the CLI path (`hermes chat -Q`).

#### Skills Hub — Blank Page

`hermes skills browse` requires `hermes` in PATH, but the launchd service ran with:
```
PATH=/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin   (missing ~/.local/bin)
```

**Fix:** Added `~/.local/bin` to the launchd plist PATH, and renamed the `product-manager`
profile (15 chars) to `product` (7 chars) so the HCI's column-based table parser doesn't
merge the profile name with the model name.

#### Session Source Labels — All "📝 Other"

CLI-parsed sessions have no `source` field. The frontend mapped `null → 'other'`,
so ALL sessions appeared under a single "Other" category with no way to filter.

**Fix:** Changed `normalizeSource()` to map `null → 'cli'`, producing distinct
"⌨️ CLI" and "weixin" filter categories. Also excluded cron sessions from the
default query to reduce clutter (`WHERE source IS NULL OR source != 'cron'`).
