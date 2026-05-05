# better-sqlite3 Apple Silicon 兼容性修复

> 工程文档 — `better-sqlite3` 架构不匹配问题的修复方案。
> 最后更新：2026-05-05 | 适用版本：v3.5.1+

---

## 问题描述

`better-sqlite3` npm 包提供了一个预编译的原生模块
（`build/Release/better_sqlite3.node`），编译目标为 `x86_64`。
在 Apple Silicon Mac（`arm64e`）上，每次调用 `new Database(path)` 都会抛出：

```
dlopen(...better_sqlite3.node):
  incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

这会破坏 HCI 中所有直接读取 Hermes `state.db` SQLite 数据库的功能。

---

## 架构

### 数据流

```
┌──────────────────────────────────────────────────────────┐
│                   HCI (Node.js)                          │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐   ┌─────────────┐  │
│  │ 原生路径      │    │ sqlite3 CLI  │   │ hermes CLI  │  │
│  │ (优先尝试)    │───→│ (降级1)      │──→│ (降级2)     │  │
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

### 调用点汇总

| # | 文件 | 函数/路由 | 原生方式 | 降级1 | 降级2 |
|---|------|----------|---------|-------|-------|
| 1 | `server.js:1384` | `loadSessionsFromDb()` | `new Database()` | `loadSessionsFromDbCli()` via `sqlite3 -json` | — |
| 2 | `server.js:4068` | `GET /api/sessions/:id/messages` | `new Database()` | `hermes sessions export` (60s超时) | — |
| 3 | `server.js:4204` | `GET /api/usage/:days` | `new Database()` | `getInsights()` via `hermes insights` | — |
| 4 | `server.js:4284` | `GET /api/usage/daily/:days` | `new Database()` | `getDailyViaCli()` via `sqlite3 -json` | — |

---

## 修复详情

### 1. 会话列表 — 消息计数 & 来源

**文件：** `server.js` — `loadSessionsFromDb()`

**修复前：** 原生 Database() 崩溃 → dbSessions 为空 → 所有会话显示 0 条消息。

**修复后：** try/catch 包裹 + `loadSessionsFromDbCli()` 降级：

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

关键改进：
- 使用 `COALESCE(source, 'cli')` 让 CLI 会话有正确的来源标签
- 排除 cron 会话（`WHERE source IS NULL OR source != 'cron'`）
- 降级仅返回 message_count + source（不含预览/最后活动时间）

### 2. 会话消息 — 点击查看

**文件：** `server.js` — `GET /api/sessions/:id/messages`

**修复前：** `new Database()` 在 try 内部崩溃 → 被 catch → 返回 `{ok: false}` → 前端显示错误。

**修复后：** 双层 try + CLI 降级：

1. 优先尝试原生 DB 读取（快速，完整元数据）
2. 失败时 → `execHermes(['sessions', 'export', ...], 60000)`：
   - 60 秒超时（大会话可能有 800+ 条消息，1.3MB JSONL）
   - 解析导出的 JSONL 为 `{id, role, content, tool_calls, timestamp}` 数组
   - **关键修复：** `tool_calls` 可能是字符串（需要 `JSON.parse`）或已是对象
     — 解析前检查 `typeof`

```js
let toolCalls = m.tool_calls || null;
if (typeof toolCalls === 'string') {
  try { toolCalls = JSON.parse(toolCalls); } catch { toolCalls = null; }
}
```

### 3. 用量概览 — 统计

**文件：** `server.js` — `GET /api/usage/:days`

**修复前：** 100+ 行复杂 DB 聚合逻辑，多次 `new Database()` 调用遍历所有
profile。第一次调用就已崩溃。

**修复后：** 将整个请求体替换为委托给已有的 `getInsights()`：

```js
const data = await getInsights(days, profile);
res.json({ ok: true, ...data, period: ... });
```

`getInsights()` 本身正常工作 — 它调用 `shell('hermes insights --days N')`，
通过 `bash -lc` 执行，总能在用户 shell PATH 中找到 `hermes`。

### 4. 用量图表 — 每日明细

**文件：** `server.js` — `GET /api/usage/daily/:days`

**修复前：** 同样 `new Database()` 崩溃 → 空响应。

**修复后：** 原生尝试 → `getDailyViaCli()` 降级：

三个独立的 `sqlite3 -json` 查询：
- **每日聚合：** `SELECT DATE(started_at, ...) as date, COUNT(*), SUM(input_tokens), SUM(output_tokens) ... GROUP BY date`
- **小时分布：** `SELECT CAST(strftime('%H', ...) AS INTEGER) as hour, COUNT(*), SUM(...) ... GROUP BY hour`
- **总计：** `SELECT COUNT(*), SUM(input_tokens), SUM(output_tokens) ...`

### 5. 发送消息 — WS 超时

**文件：** `src/js/main.js` — `sendViaWebSocket()`

**修复前：** WS 断开时 Promise 永不 resolve → CLI 降级永远不执行。

**修复后：** 添加 15 秒超时：

```js
const wsTimeout = setTimeout(() => {
  wsClient.removeEventListener('message', onDone);
  reject(new Error('WebSocket timeout (15s)'));
}, 15000);
```

成功/失败/未连接时清除超时。

---

## PATH 要求

以下二进制文件必须在 launchd PATH 中 HCI 才能正常运行：

| 二进制 | 默认位置 | 用于 |
|--------|---------|------|
| `hermes` | `~/.local/bin/hermes` | `execHermes()`, `shell()` |
| `sqlite3` | `/usr/bin/sqlite3` | `loadSessionsFromDbCli()`, `getDailyViaCli()` |

**launchd plist 修复 (`com.hermes.control-interface.plist`)：**

```xml
<key>PATH</key>
<string>/Users/chengjiahui/.local/bin:/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
```

---

## 影响分析

| 指标 | 修复前（仅原生） | 修复后（含降级） |
|------|-----------------|------------------|
| 会话列表加载 | 崩溃 → 空 | 完整列表含消息计数 |
| 聊天消息 | 崩溃 → 错误 | 完整消息历史（通过 export） |
| 用量概览 | 崩溃 → 空白 | 完整统计（通过 hermes insights） |
| 每日图表 | 崩溃 → 空白 | 每日 + 小时聚合 |
| 发送消息 | WS 挂死 → 无响应 | WS 超时 + CLI 降级 |

---

## 上游更新后恢复

```bash
# 1. 拉取上游更新
git pull origin main

# 2. 重打兼容补丁
git am patches/better-sqlite3-compat.patch
# 如有冲突：手动解决后 git am --continue

# 3. 恢复 launchd PATH
# 编辑 ~/Library/LaunchAgents/com.hermes.control-interface.plist

# 4. 重建前端 + 重启
npm run build
launchctl stop com.hermes.control-interface
launchctl start com.hermes.control-interface
```

如果补丁无法干净应用，使用交互方式：
```bash
# 将修复保存为分支，然后 cherry-pick
git checkout -b fix/better-sqlite3-compat
git rebase main
# 解决冲突，然后：
git checkout main
git merge fix/better-sqlite3-compat
```
