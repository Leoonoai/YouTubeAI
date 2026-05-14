---
id: lesson-2026-04-task-misplacement-state-drift
title: 任务错放与状态脱节——两类反模式与机制化防御
status: active
confidence: 85
tags:
  - task-workflow
  - task-hierarchy
  - plan-workflow
  - git-hooks
createdAt: 2026-04-09T00:00:00.000Z
sourceTasks:
  - '108'
  - '109'
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 两类反模式

本 lessons 记录在 108 号任务里发现、在 109 号任务里设计防御的两类反模式。

### 1. 任务错放（cross-project misplacement）

**症状**：任务文件躺在 `.imoss/tasks/active/` 下，但内容完全属于某个子项目（例如 IMOSS 106 "博客系统——前台博客页面+后台内容管理"，内容是为 onoai 写的，和 onoai 010 完全重复）。

**根因**：
- `scripts/task-create.mjs` 的项目落点逻辑只看 `active-session.json` 和 cwd，不参考标题语义
- 自然语言触发 "新建任务 xxx" 时，agent 往往直接调脚本不传 `--project`，当前 session 绑定哪个项目就落到哪个
- 即使标题里明显写着"onoai 博客系统"也不会被拦截

**为什么会写成这样**：开发 task-create.mjs 时默认场景是"在对应项目的窗口里创建任务"，没考虑到"用户在 IMOSS 窗口里用自然语言替 onoai 创建任务"的路径。

### 2. 状态脱节（plan-code drift）

**症状**：
- `git log` 里已经有 `feat(NNN): ...` commit
- 对应任务的 `plan.md` 全是 TODO 模板占位
- frontmatter `status` 还在 `investigating`

**真实案例**：
- 104 "Dashboard 首页增加系统资源监控"：commits `2438977` + `420c35b` 已完整实现，但 plan 全 TODO
- 107 "扩展资源监控+内存释放功能"：commit `89e3304` 已落地主要功能，但 plan 全 TODO

**根因**：
- agent 绕过 investigate→draft→plan 四段式工作流，直接写代码再 commit
- 098 号任务加的 hook 只拦"Claude 在 medium/large 任务投入 investigating/planning 阶段时修改源码"，但：
  - 纯 Codex 环境所有 Claude hook 都不触发
  - hook 不拦 `git commit` 本身，只拦源码写入
  - agent 可以在 plan 还是 TODO 的情况下先让 status = `doing`，然后合法写代码
- commit 后没有任何机制校验"feat(NNN) 是否对应一个 plan 已完成的任务"
- commit message 惯用 `feat(模块):` 而非 `feat(NNN):`，事后反查任务号非常困难（104 的 commit 用了 `feat(dashboard)`）

## 防御机制（109 落地）

### A. 创建期拦截错放

`scripts/task-create.mjs` 现在会做"项目意图检测"：
- 扫描 title 是否命中已注册子项目名或别名（onoai / baima / rephoto / openclawzs，以及 `baima → baima-juben`、`openclaw → openclawzs` 这类常见写法）
- 若命中且当前落点项目不同 → 拒绝静默创建，要求二选一：
  - `--project <name>` 明确目标项目
  - `--force-project-mismatch` 承认是 IMOSS 基础设施任务

### B. commit 后审计脱节

`scripts/git-hooks/post-commit`（新增）在每次 commit 后调用 `scripts/hooks/post-commit-audit.mjs`：
- 解析 commit subject 的 `feat(NNN)` / `fix(NNN)` / `chore(NNN)` / `imoss(NNN)` 前缀
- 在 IMOSS 和所有子项目的 `tasks/active+done/` 里定位对应任务目录
- 校验：
  - status 是否仍在 `investigating` / `planning` → 告警
  - plan.md 是否仍含 `## 目标 / ## 步骤 / ## 完成标准 + TODO:` 模板占位 → 告警
  - medium/large 任务且 status ≥ doing 时 review-log 是否 accepted → 告警
- **非阻塞**：告警后 hook 仍 `exit 0`，不破坏 commit 体验

安装通过 `scripts/install-git-hooks.mjs` 管理（把 `scripts/git-hooks/*` 同步到 `.git/hooks/*`），并集成进 `scripts/setup.mjs`。

### C. Dashboard 健康信号

`dashboard/server/routers/search.ts` 新增 `taskHealth` tRPC 接口：
- 扫描所有 `lifecycle=active` 的任务
- 返回 `templateTasks`（plan 仍是 TODO 模板的）和 `stuckTasks`（在 investigating/planning 卡 >3 天的）

`SystemHealth` widget 接入，在健康面板里暴露两类新异常，source 标记为 `109 health`，扣分进健康分计算。

### D. 规则与文案收敛

- `.imoss/rules/core/task-hierarchy.md` 增补"任务归属判断"和"commit message 与任务号约定"章节
- `CLAUDE.md` 的"Hook 强制执行"节把 `executing` 改为 `doing`，与真实状态机对齐
- `optional/hooks/claude/user-prompt-submit.cjs` 同步术语，不再提示 agent "改为 executing"

## 关键取舍

- **不做自动推进 plan 状态**：`currentTaskId` 虽然在 task-create 时会写回 active-session，但让 commit 触发 `investigating → doing` 的"魔法联动"代价太高（误推会掩盖真实漏填），只做非阻塞告警
- **不做阻塞式 commit-msg 拦截**：会让用户绕过（`--no-verify`），破坏信任基础
- **方案必须同时覆盖 Claude 和纯 Codex**：关键防线落在 repo 脚本 + git hook，不能只依赖 Claude hook
- **先命令行/仓库闭环，再 Dashboard 可见化**：防御优先于观测

## 如何判断任务归属（对未来 agent）

| 类型 | 归属 | 举例 |
|---|---|---|
| 产品功能 / 业务逻辑 / 业务数据 | 对应子项目 | onoai 博客、baima-juben 剧本 |
| IMOSS 基础设施 / 工具 / 脚本 | IMOSS | task-create 脚本、Dashboard widget、git hook、rule 文档 |
| 跨项目通用能力 | IMOSS | Telegram channel 配置、tmux session 管理 |
| 某个项目的 tmux/telegram 接入 | IMOSS | 实现落在 `.imoss/tmux-sessions.json`、`scripts/restart-project-window.mjs` |

**判断技巧**：问"这改动会落在 `projects/<name>/` 目录里，还是落在 IMOSS 根目录里？"前者归子项目，后者归 IMOSS。

## 参考

- `.imoss/tasks/done/108-*/plan.md` — 症状清理任务
- `.imoss/tasks/active/109-防止任务错放与状态脱节根因分析与机制设计/` — 防御机制实现
- `scripts/task-create.mjs` L134-189 — 项目意图检测
- `scripts/git-hooks/post-commit` + `scripts/hooks/post-commit-audit.mjs` — commit 审计
- `dashboard/server/routers/search.ts` `taskHealth` — Dashboard 健康信号源
- `.imoss/rules/core/task-hierarchy.md` "任务归属判断" 节 — 规则条文

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-04-09-subproject-hook-gap|109 post-commit 审计未覆盖子项目 git——onoai 实锤脱节案]]
