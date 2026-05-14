---
id: decision-2026-05-imoss-devresearch-架构整合路线
title: >-
  IMOSS dev/research 体系整合走 UI 层合并 + 复用 171 task-bound 路线，不删 /research 也不扩
  window-register
status: active
confidence: 90
tags:
  - architecture
  - dashboard
  - research-system
  - tmux
  - window-management
source: task-241
createdAt: 2026-05-06T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

把"开发任务"和"研究课题"在 UI/认知层合并成一套，但分 5 步走小改动，不破现有结构：

1. **UI 合并**：保留 /tasks 和 /research 两个菜单，加 segmented filter "开发/研究/全部"，首页加"全部活跃任务"flat union 区。**不删** /research 详情视图（235 的阶段摘要+结论 tab 不可复用到 dev task）。
2. **多研究 tmux 窗口**：**复用 171 `codex-agent-start.mjs --task <id>` 的 task-bound 窗口**，不扩 `window-register.mjs` 三维 schema。
3. **跨对话延续**：复用现有 `active-session` + plan/progress.md，**不**加 research 专属 session 字段。Claude 改 `session-start.cjs`，Codex 改 `codex-bootstrap.cjs`（两条恢复路径分开）。
4. **cron 自动迭代**：dashboard scheduler 直接 `tmux send-keys -t <window> -l "迭代 NNN"`（自然语言伪用户输入），不发不存在的 slash command；保留 marker + Telegram 兜底。
5. **完成事件**：augment skill iterate 完成时 `writeEvent('research_iteration_done', {taskId, sourceWindow, finishedAt, refDelta, lessonDelta})` 到 `.imoss/events.jsonl`；主控窗口下次 prompt 时 hook 注入"task NNN X 分钟前完成"。

## 背景

225-237 已经把研究系统建好（`taskType: research` + sourceTasks frontmatter + cron + marker queue + subagent 下沉）。但用户体感 dev/research 仍是两套——dashboard 上 /tasks 和 /research 是分页面，且单对话内 subagent 隔离 ≠ 跨对话长期推进。241 调研后明确两层痛点：

1. **认知层**：UI 把它们分成两块；底层 task 已统一只是看不出来
2. **工程层**：subagent 一次性，关掉就没了，需要让 2-3 个深度研究课题各跑一个 tmux 窗口，跨对话续

不需要 hermes 化 standalone runtime（量级 2-3 不到），但需要"任务粒度的窗口绑定"。

## 理由

1. **不删 /research 路由**：235 的阶段摘要+结论 tab+原文 tab 是研究详情专门做的，强行 union 到 dev task 详情会让两边都变重，分开维护漂移更小。
2. **复用 171 task-bound 而非扩 window-register 三维**：Codex 调研发现 `restart-project-window.mjs` 设计上会清理 per-window active-session（"重启回 lobby"），跟"重启恢复 task 绑定"语义冲突。171 的 `codex-agent-start` 已经做了 task-bound active-session 写入 + bootstrap + registry + sweep，PoC 阶段直接复用比改 project window registry 安全。如果将来真有 Claude/TG 研究窗口需求，再抽 generic `task-agent-start --executor claude|codex`。
3. **send-keys 自然语言而非 slash command**：Claude Code 安全模型要求 hook 触发必须来自"用户主动发消息"。dashboard 进程同用户身份能 `tmux send-keys`，等价"伪用户输入"。234 的 `RESEARCH_ITERATE_COMMAND` hook 已经能识别"迭代 NNN"字面量，复用即可，不要发不存在的 `/iterate` slash command。
4. **events.jsonl 用现有 `writeEvent` helper**：append-only，`writeEvent` 自动加的 `timestamp` 是 helper 元字段，业务字段（如 `finishedAt`）必须显式写。2-3 窗口规模够用；>5 再补统一 `appendJsonlEvent` 提供严格事务语义。
5. **per-window seen state 去重 hook 注入**：events.jsonl 是 append-only 协作信号没有 exactly-once；hook 注入完成提示要写 `.imoss/sessions/research-iteration-done-seen-<sessionKey>.json` 记录 lastSeen，避免主控窗口 5 分钟内每次 prompt 都重复刷"task NNN 完成"。

## 操作规范

落地按 5 个 dev task 顺序，每个独立 plan + 独立 codex review + 独立 commit：

| 顺序 | task | scope | 改动量 | 价值 |
|---|---|---|---|---|
| 1 | 242 UI 合并：Tasks filter + Home union + mdComponents 共享 | medium | ~250 行 | 用户体感"一套" |
| 2 | 243 codex-bootstrap research 块 + agentKind 标记 | medium | ~118 行 | 启用研究窗口长期运行 |
| 3 | 244 scheduler 直注入 + events.jsonl + hook 主控感知 | medium | ~210 行 | cron 自动迭代闭环 |
| 4 | 245 Home 卡片补 living 上次/下次 | small | ~30 行 | 体验完成度收尾 |
| 5 | （可选）抽 generic `task-agent-start --executor claude|codex` | medium | ~150-250 行 | 仅 PoC 证明需要 Claude/TG 研究窗口才做 |

## 参考

- task 241 plan.md（5 个问题域 A/B/C/D/E 完整方案 + 风险表）
- [[2026-05-06-lesson-react-hooks-多项目-trpc-查询-task242]]
- [[2026-05-06-lesson-yaml-sourceTasks-双格式解析-task243]]
- [[2026-05-06-decision-tmux-send-keys-自然语言伪用户输入-task244]]
- 171 codex-agent-start / 234 living research cron / 235 /research 详情页 / 237 subagent 下沉

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-06-decision-tmux-send-keys-自然语言伪用户输入-task244|dashboard scheduler 用 tmux send-keys 注入自然语言"迭代 NNN"绕过 cron→agent 限制；events.jsonl 业务字段不依赖 helper 元字段]]
- [[2026-05-06-decision-研究执行下沉-subagent-task237|研究课题的 search→fetch→distill 循环下沉到 Agent 工具的 subagent 执行，让多课题 context 天然隔离]]
- [[2026-05-07-decision-codex-worker-接管研究执行-task254|IMOSS 研究执行 worker 从主控 Claude 委派给 Codex 窗口（v5 路径 A）]]
- [[2026-05-07-decision-v4-pkm-闭环架构-task246-251|IMOSS 自我研究成长系统 v4 架构 — 单 worker 串行 + 建议池统一收口 + 持续追新闭环]]
