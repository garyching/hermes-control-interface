# 更新日志

> 中文版 · [English](CHANGELOG.md)

所有重要变更均记录在此文件。

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，
版本号遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

---

## [3.5.1] — 2026-05-05

### 修复

#### better-sqlite3 架构兼容性（Apple Silicon）

**问题：** `better-sqlite3` 原生模块是 `x86_64` 架构编译的，当前设备为 `arm64e`
（Apple Silicon Mac）。每次调用 `new Database()` 都会抛出：
```
dlopen(...better_sqlite3.node): incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')
```

**解决方案：** 三层降级策略 — 先试原生模块，失败降级到 `sqlite3` CLI，再失败降级到 `hermes` CLI。
覆盖全部 4 个数据库访问点：

| 接口 | 问题 | 修复方式 |
|---|---|---|
| 会话列表 — 消息数始终 `0` | `loadSessionsFromDb` 崩溃 | 新增 `loadSessionsFromDbCli()` 通过 `sqlite3 -json` |
| 聊天 — 点会话无消息 / 挂死 | `/api/sessions/:id/messages` 崩溃 | 新增 `hermes sessions export` JSONL 降级 |
| 用量 — 标签无数值 | `/api/usage/:days` 崩溃 | 委托给 `getInsights()`（`hermes insights` CLI） |
| 用量图表 — 空白 | `/api/usage/daily/:days` 崩溃 | 新增 `getDailyViaCli()` 通过 `sqlite3 -json` |

完整文档：`docs/BETTER_SQLITE3.zh-CN.md` | 可重放补丁：`patches/better-sqlite3-compat.patch`

#### 聊天发送 — 无响应

WebSocket 连接不稳定（频繁 ping 超时）。`sendViaWebSocket()` 创建了一个
永不 resolve/reject 的 Promise，导致 CLI 降级路径永远无法执行。

**修复：** 给 WebSocket 发送路径加了 15 秒超时。超时后 Promise reject，
自动降级到 `hermes chat -Q` CLI 模式。

#### Skills Hub — 空白页

`hermes skills browse` 需要 `hermes` 在 PATH 中，但 launchd 服务的 PATH 为：
```
PATH=/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin   （缺少 ~/.local/bin）
```

**修复：** 在 launchd plist 的 PATH 中加入 `~/.local/bin`；同时将 `product-manager`
（15 字符）重命名为 `product`（7 字符），避免 HCI 表格解析器将 profile 名与模型名合并。

#### 会话来源标签 — 全部显示 "📝 Other"

CLI 解析的会话没有 `source` 字段。前端将 `null → 'other'`，导致所有会话
堆在一个 "Other" 类别下，无法过滤。

**修复：** 将 `normalizeSource()` 改为 `null → 'cli'`，产生 "⌨️ CLI" 和 "weixin"
两个独立过滤分类。同时在 DB 查询中排除 cron 会话以减少干扰
（`WHERE source IS NULL OR source != 'cron'`）。
