# 问题目录

> 中文版 · [English](ISSUES.md)
>
> HCI 部署过程中发现的所有问题，按严重度排列。
> 每个问题关联修复 commit 和影响文件。

| # | 问题 | 严重度 | 修复 Commit | 组件 |
|---|------|--------|------------|------|
| 1 | `better-sqlite3` 架构不匹配 | 🔴 致命 | `2f43cb5` | 数据库层 |
| 2 | 会话列表 — 消息数全为 0 | 🔴 致命 | `2f43cb5` | 聊天页 |
| 3 | 聊天 — 点击会话无消息 | 🔴 致命 | `2f43cb5` | 聊天页 |
| 4 | 聊天 — 发送消息无响应 | 🔴 致命 | `2f43cb5` | 聊天页 |
| 5 | 用量页 — 标签无数值 | 🔴 致命 | `2f43cb5` | 用量页 |
| 6 | 用量图表 — 空白 / 无数据 | 🔴 致命 | `2f43cb5` | 用量页 |
| 7 | Skills Hub — 空白 / "No skills found" | 🟡 严重 | — | Skills页 |
| 8 | 会话列表 — 全部 "📝 Other"，无法过滤 | 🟡 严重 | `2f43cb5` | 聊天侧栏 |
| 9 | Cron 会话淹没列表 | 🟢 次要 | `2f43cb5` | 聊天侧栏 |
| 10 | Agent 名称 — 模型名混入标题 | 🟢 次要 | `2f43cb5` | Agents页 |

---

## 问题 1：better-sqlite3 架构不匹配

**严重度：** 🔴 致命 | **组件：** 数据库层  
**影响：** 所有读取 Hermes `state.db` 的页面  
**修复 Commit：** `2f43cb5`

### 描述

`better-sqlite3` npm 包提供预编译原生模块，编译目标为 `x86_64`。
在 Apple Silicon Mac（`arm64e`）上，每次 `new Database()` 调用抛出：

```
dlopen(...better_sqlite3.node):
  incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

这是**问题 2–6 的根本原因**。所有下游修复均实现三层降级策略：
原生 → `sqlite3` CLI → `hermes` CLI。

### 修改文件

| 文件 | 函数/路由 | 变更 |
|------|----------|------|
| `server.js:1384` | `loadSessionsFromDb()` | 添加 try/catch + `loadSessionsFromDbCli()` |
| `server.js:1400` | `loadSessionsFromDbCli()` | 新函数 — `sqlite3 -json` 查询 |
| `server.js:4068` | `GET /api/sessions/:id/messages` | 添加 CLI export 降级 |
| `server.js:4204` | `GET /api/usage/:days` | 替换为 `getInsights()` 委托 |
| `server.js:4284` | `GET /api/usage/daily/:days` | 添加 `getDailyViaCli()` 降级 |
| `server.js:4400` | `getDailyViaCli()` | 新函数 — 3 个 `sqlite3 -json` 查询 |

### 恢复

```bash
git am patches/better-sqlite3-compat.patch
```

---

## 问题 2：会话列表 — 消息数全为 0

**严重度：** 🔴 致命 | **组件：** 聊天页（侧栏）  
**依赖：** 问题 1 (better-sqlite3)  
**修复 Commit：** `2f43cb5`

### 症状

聊天侧栏中每个会话都显示 "0 msgs"，与实际消息数无关。

### 根因

`loadSessionsFromDb()` 使用 `new Database()` 崩溃（问题 1）。
缺少 DB 数据，`message_count` 字段永远为空，前端默认显示 0。

### 修复

新增 `loadSessionsFromDbCli()` — 使用 `sqlite3 -json` 直接查询
`message_count`。同时返回 `source` 字段用于过滤分类。

---

## 问题 3：聊天 — 点击会话无消息

**严重度：** 🔴 致命 | **组件：** 聊天页（消息查看器）  
**依赖：** 问题 1 (better-sqlite3)  
**修复 Commit：** `2f43cb5`

### 症状

点击侧栏中任何会话会选中它（📌），但主区域仍显示
"Welcome to Chat"，没有消息历史。用户看不到任何错误提示。

### 根因

`GET /api/sessions/:id/messages` 使用 `new Database()` 崩溃（问题 1）。
catch 块返回 `{ok: false, error: ...}`，前端不显示该错误 —
只是停留在欢迎界面。

此外，初始降级代码有一个 bug：`tool_calls` 字段可能是字符串
（需要 `JSON.parse`）或已是解析好的对象。对对象执行 `JSON.parse()` 抛出异常，
而静默 catch 吞掉了所有消息。

### 修复

1. 优先尝试原生 DB 读取
2. 失败时 → `execHermes(['sessions', 'export', ...], 60000)` 带 60 秒超时
3. 解析 JSONL：从每个导出会话对象中提取 `messages` 数组
4. **关键：** `JSON.parse` 前检查 `typeof toolCalls === 'string'`

---

## 问题 4：聊天 — 发送消息无响应

**严重度：** 🔴 致命 | **组件：** 聊天页（输入框）  
**修复 Commit：** `2f43cb5`

### 症状

输入消息并按回车/发送后无响应。用户消息出现在聊天区域，
但助手从不回复。没有显示错误。

### 根因

`sendChatMessage()` 优先尝试 WebSocket。WS 连接不稳定
（频繁 `[WS] ping timeout, forcing reconnect`）。
当 `wsClient.connected` 为 `true` 时，调用 `sendViaWebSocket()`，
它创建的 Promise 在 WS 断开后**永不 resolve 或 reject**。
CLI 降级代码（`sendViaCLI`）永远无法到达。

### 修复

在 `sendViaWebSocket()` 中添加 15 秒超时：

```js
const wsTimeout = setTimeout(() => {
  wsClient.removeEventListener('message', onDone);
  reject(new Error('WebSocket timeout (15s)'));
}, 15000);
```

超时触发后，Promise reject → `catch(wsErr)` 记录警告 →
降级到 `sendViaCLI()` 执行 `hermes chat -Q`。

### 修改文件

| 文件 | 函数 | 变更 |
|------|------|------|
| `src/js/main.js:1771` | `sendViaWebSocket()` | 添加 15s 超时，所有路径清除超时 |

---

## 问题 5：用量页 — 标签无数值

**严重度：** 🔴 致命 | **组件：** 用量页  
**依赖：** 问题 1 (better-sqlite3)  
**修复 Commit：** `2f43cb5`

### 症状

用量页显示列标题（"Sessions"、"Messages"、"Input Tokens"等）
但所有数值为空白/零。
Agent 过滤下拉框只显示 "All agents"（无具体 profile）。

### 根因

`GET /api/usage/:days` 包含 100+ 行复杂 DB 聚合逻辑，
多次 `new Database()` 调用。第一次调用崩溃（问题 1），
catch 块返回 `{ok: false, error: ...}`。

### 修复

将整个请求体替换为单行委托给 `getInsights()`：

```js
const data = await getInsights(days, profile);
res.json({ ok: true, ...data, period: ... });
```

`getInsights()` 正常工作，因为它调用 `shell('hermes insights --days N')`，
通过 `bash -lc` 执行，总能在用户 shell PATH 中找到 `hermes`。

---

## 问题 6：用量图表 — 空白 / 无数据

**严重度：** 🔴 致命 | **组件：** 用量页（图表）  
**依赖：** 问题 1 (better-sqlite3)  
**修复 Commit：** `2f43cb5`

### 症状

"Daily Token Trend"、"Daily Cost"和"Model Distribution"的 canvas 区域空白。
"Models"、"Platforms"部分显示 "No data"。

### 根因

`GET /api/usage/daily/:days` 通过 `new Database()` 查询 `state.db`，
崩溃（问题 1）。

### 修复

新增 `getDailyViaCli()` — 3 个独立的 `sqlite3 -json` 查询：

1. **每日聚合** — `SELECT DATE(started_at, ...) as date, COUNT(*), SUM(...) GROUP BY date`
2. **小时分布** — `SELECT CAST(strftime('%H', ...) AS INTEGER), COUNT(*), SUM(...) GROUP BY hour`
3. **总计** — `SELECT COUNT(*), SUM(input_tokens), SUM(output_tokens), SUM(message_count)`

返回 `byModel: []` 和 `byPlatform: []`（CLI 无法获取这部分数据），
因此 Models/Platforms 列表显示 "No data" 为已知限制。

---

## 问题 7：Skills Hub — 空白 / "No skills found"

**严重度：** 🟡 严重 | **组件：** Skills 页

### 症状

Skills Hub 显示页头和搜索栏，但技能列表显示
"No skills found on page 1"。点击刷新无效。

### 根因

`execHermes(['skills', 'browse', '--page', '1'])` 直接调用
`execFile('hermes', ...)`（无 bash 包装）。launchd 以有限 PATH 启动 HCI：
```
/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin
```
`hermes` 位于 `~/.local/bin/hermes`，**不在** launchd 的 PATH 中。

`shell()` 函数正常工作（使用 `bash -lc`），但 `execHermes()` 不行。

### 修复

在 launchd plist 中添加 `~/.local/bin`：

```xml
<key>PATH</key>
<string>/Users/chengjiahui/.local/bin:/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
```

### 修改文件

| 文件 | 变更 |
|------|------|
| `~/Library/LaunchAgents/com.hermes.control-interface.plist` | 添加 PATH 条目 |

---

## 问题 8：会话列表 — 全部 "📝 Other" / 无过滤选项

**严重度：** 🟡 严重 | **组件：** 聊天侧栏  
**修复 Commit：** `2f43cb5`

### 症状

来源过滤下拉框只显示 "All" 和 "📝 Other" —
无法按 CLI、Weixin 或 cron 过滤。

### 根因

`normalizeSource(s)` 将 `null` 来源映射为 `'other'`。
CLI 解析的会话没有 `source` 字段（CLI 表格输出不包含来源列），
因此每个会话都得到 `_source = 'other'`。

### 修复

修改 `normalizeSource()`：
```js
// 修复前
if (!src) return 'other';
// 修复后
if (!src) return 'cli';
```

同时在 `loadSessionsFromDbCli()` SQL 查询中添加 `COALESCE(source, 'cli')`。

---

## 问题 9：Cron 会话淹没聊天列表

**严重度：** 🟢 次要 | **组件：** 聊天侧栏  
**修复 Commit：** `2f43cb5`

### 症状

数百条 cron 任务会话（标题如 "cron_..."）填满聊天侧栏，
将用户主动发起的对话挤到列表底部甚至不可见。

### 根因

默认 SQL 查询获取所有会话，包括 cron 任务。由于 cron 每几分钟执行一次，
有较新的 `ended_at` 时间戳，排在最前面。

### 修复

在 `loadSessionsFromDbCli()` 中添加 `WHERE source IS NULL OR source != 'cron'`。

---

## 问题 10：Agent 名称 — 模型名混入标题

**严重度：** 🟢 次要 | **组件：** Agents 页

### 症状

在 Agents 页上，profile `product-manager` 搭配模型 `ark-code-latest` 显示为：
```
product-manager ark-code-latest Status ○ Stopped Model stopped
```
模型名合并到了 profile 名称字段中。

### 根因

HCI 表格解析器使用 `split(/\s{2,}/)` 分隔列。profile 名称
`product-manager`（15 字符）和模型 `ark-code-latest`（15 字符）
之间只有 1 个空格（因为 profile 几乎填满列宽）。正则要求 2+ 空格，
因此分割将两者合并。

### 修复

将 profile 从 `product-manager` 重命名为 `product`（7 字符），
确保模型名前至少有 8 个空格。

### 修改文件

无（仅 CLI 重命名）：
```bash
hermes profile rename product-manager product
```
