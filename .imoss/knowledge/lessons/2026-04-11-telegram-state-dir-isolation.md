---
id: lesson-2026-04-11-telegram-state-dir-isolation
title: Telegram plugin 共享默认 state dir + last-restart-wins + wrapper bash env 丢失三重坑
status: active
confidence: 90
tags:
  - telegram
  - tmux
  - env-vars
  - hooks
  - state-dir
  - plugin-isolation
createdAt: 2026-04-11T00:00:00.000Z
sourceTasks:
  - '129'
relatedLessons:
  - 2026-04-09-subproject-hook-gap
fixedIn:
  - commit: 80e247f
    date: 2026-04-11T00:00:00.000Z
    summary: telegram 频道按 tmux 窗口名隔离 + per-window 会话态清理
  - commit: 6bec99d
    date: 2026-04-11T00:00:00.000Z
    summary: wrapper bash env 持久化（export K=V; cmd），修项目绑定跨窗口污染回归
  - commit: e99b274
    date: 2026-04-11T00:00:00.000Z
    summary: tmux-telegram 规则加 §0 铁律，固化三层解耦
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 事件

2026-04-11 上午用户报告 IMOSS 窗口对外 Telegram bot 断线。查 `~/.claude/channels/telegram/bot.pid` 指向的 bun 进程跑的实际上是 baima 的 token（`8278340195...`），不是 imoss 的 `8288513407...`。telegram 默认槽被 baima 抢了。

按"one-window-one-bot" 修完 state dir 隔离，Step 5 回归测试时再踩两个二级坑：
1. 新开的 onoai/openclawzs/baima 窗口 session-start hook 打出 `[PROJECT MODE - explicit] bound to P005 baima-juben` —— 跨窗口项目污染
2. 用户从 Telegram 发"你好"，imoss bot 正确接收，但 imoss claude 回复"当前窗口已绑定 P005 baima-juben"

## 根因（三层坑叠加）

### 坑 1：Telegram plugin 的默认 state dir 共享 + last-restart-wins

- `~/.claude/plugins/.../telegram/0.0.5/server.ts` 的 `STATE_DIR` fallback 是写死的 `~/.claude/channels/telegram/`
- plugin 启动时会对已有 `bot.pid` 的进程发 `SIGTERM`，不校验是不是同一个 token
- 任何没注入 `TELEGRAM_STATE_DIR` 的窗口都用默认槽 → 谁后重启谁抢占

### 坑 2：`readActiveSession` 的 legacy fallback 跨窗口串读

- `optional/hooks/lib/session-key.cjs` 最初设计是"per-window 文件找不到就 fallback 到 legacy `.imoss/active-session.json`"，兼容老版本
- 问题：一个 stable-key 窗口（有 `IMOSS_SESSION`）在 per-window 文件被 restart 脚本清掉之后，会读到其他窗口刚写的 legacy，打出完全错误的 `[PROJECT MODE - explicit]` 绑定
- session-end handoff writer 的 `getCurrentTaskId` 走同一条 fallback → 把 imoss 的 currentTaskId（129）串到其他窗口的 handoff 文件

### 坑 3：wrapper bash 的 inline env 丢失（最隐蔽）

- `scripts/restart-project-window.mjs` 原本用 shell inline 赋值启动 claude：
  ```bash
  bash -c "IMOSS_SESSION=imoss TELEGRAM_BOT_TOKEN=xxx TELEGRAM_STATE_DIR=yyy claude --dangerously-..."
  ```
- **inline 赋值只对紧随其后的那一条命令生效**，不进入 wrapper bash 本身的 env 表
- 后果链：
  1. 第一次 claude 启动 → 有 env → 正确
  2. claude 退出（/exit、/clear、context 压缩重启、crash）
  3. wrapper 里 `exec bash -i` 接管 → 新交互 bash **没有** `IMOSS_SESSION` / `TELEGRAM_*`
  4. 用户或 Dashboard "重启进程"按钮在这个 bash 里再跑 `claude` → 新 claude **也没有** stable session key
  5. `hasStableSessionKey()=false` → `readActiveSession` **走 legacy fallback**
  6. legacy 里是别人最后写的内容 → 读到 baima 的 P005 绑定
- 更糟的是：坑 2 的 fix 靠 `hasStableSessionKey()==true` 时不 fallback legacy 保护，而这个场景**真的没有** stable key，保护条件被合法绕过

## 修复

### Fix 1（state dir 隔离）
- `scripts/restart-project-window.mjs`：如果 `entry.channel` 以 `telegram` 开头，自动注入 `TELEGRAM_STATE_DIR=$HOME/.claude/channels/<entry.channel>`
- `.imoss/tmux-sessions.json`：imoss 条的 `channel` 从 `telegram` 改成 `telegram-imoss`（不再占默认槽）
- 新建 `~/.claude/channels/telegram-imoss/` 和 `telegram-baima/`，把非密钥状态（`access.json` / `approved/`）迁过去；`.env` **不动**（安全规则禁止 agent 读写 `.env`）

### Fix 2（legacy fallback 收口）
- `optional/hooks/lib/session-key.cjs` 的 `readActiveSession` 在 `hasStableSessionKey()==true` 时直接返回 null，不 fallback legacy
- 只有 `getSessionKey()=='global'`（完全没设 `IMOSS_SESSION`/`WT_SESSION` 的老兼容窗口）才继续读 legacy
- 同一修复连带解决 session-end handoff writer 跨窗口串 taskId

### Fix 3（wrapper bash env 持久化）
- `scripts/restart-project-window.mjs` 的 envPrefix 从 `K=V K=V cmd` 改成 `export K='V'; export K='V'; cmd` 序列
- export 写进 wrapper bash 自己的 shell env 表 → `exec bash -i` 后的交互 shell 通过**进程继承**拿到 → 里面再 spawn 的 claude 也继承
- 日志脱敏正则适配新 format：`export TELEGRAM_BOT_TOKEN='...'` 替换成 `export TELEGRAM_BOT_TOKEN=<redacted>`
- 本地用 `true` 模拟 claude 退出，`/proc/<wrapper-pid>/environ` 显示三个 env 都在，实锤修复

## 教训要点

1. **Plugin state dir 默认共享是反模式**：任何持 lock 的 plugin，默认 state dir 一定要挂在某个 per-window 标识下（窗口名最稳），不要用 plugin 名做默认 dir。last-wins 抢占的调试成本很高。

2. **Wrapper bash 启动 agent 一定要 `export`，不要 inline**：`FOO=bar cmd` 只管一条命令；`export FOO=bar; cmd` 才把 env 写进 bash 自己的 env 表，后续 `exec bash -i` 和所有子进程都继承。所有通过 tmux 新开 bash 再跑长生命周期 agent 的脚本都会踩这个。

3. **Fallback 路径是"贴心功能"也是"污染管道"**：`readActiveSession` 的 legacy fallback 设计初衷是"找不到 per-window 就兼容老行为"，但在**多窗口并发 + 状态会被清理重建**的场景下，fallback 变成"读到其他窗口的最新状态"。Fallback 条件必须严格到"我真的知道自己没有绑定"，不能只是"我当前文件不存在"。

4. **tmux 窗口名 ↔ plugin channel ↔ IMOSS 项目 是三个独立维度**，不要硬绑：
   - 窗口名 ↔ Telegram state dir：**静态 1:1 永久**（落在 `entry.channel`）
   - 窗口 ↔ 项目：**动态运行时绑定**（由 `enter-project` 写入 per-window `active-<name>.json`）
   - 窗口名 ↔ 启动时默认项目：**软建议**（`entry.project` 只作为启动提示，不做硬绑定）
   历史遗留的 `tmux-sessions.json` 里 `project: P001` 字段含义被降级，`.imoss/rules/core/tmux-telegram.md` §0 已固化成铁律

5. **排查多窗口 env 丢失，直接读 `/proc/<pid>/environ`** 是最快的路径：`tr '\0' '\n' < /proc/<bash-pid>/environ | grep TELEGRAM`。不要信 shell 提示符或 agent 自报的绑定状态。

## 验证证据

- 4 个 bun server.ts 同时运行（pid 1819820 / 1804382 / 1804755 / 1815222），各自独立 `bot.pid` 在 `telegram-{imoss,baima,onoai,openclawzs}/`，互不干扰
- 端到端实证：用户从 Telegram 向 imoss bot 发消息，imoss claude 正确识别"当前是 imoss lobby"（而非此前的错误 P005 绑定）
- `/proc/<wrapper-pid>/environ` 在 Fix 3 落地后显示 `IMOSS_SESSION` / `TELEGRAM_BOT_TOKEN` / `TELEGRAM_STATE_DIR` 三项都在

## 后续

- **未做**：token 从 `tmux-sessions.json` 的 `entry.env` 迁移到 state dir `.env` 做单一事实源 —— 当前规则禁止 agent 读写 `.env`，需要单独任务 + 人工密钥处理
- **未做**：删除历史 `~/.claude/channels/telegram/` 默认槽 —— 留做向后兼容，不动它就不会坏

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-04-11-imoss-root-vs-project-task-root|Session registry root ≠ active project task root — cross-project current-task resolver]]
