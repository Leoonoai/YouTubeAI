---
id: decision-2026-05-v4-pkm-闭环架构
title: IMOSS 自我研究成长系统 v4 架构 — 单 worker 串行 + 建议池统一收口 + 持续追新闭环
status: active
confidence: 88
tags:
  - pkm
  - architecture
  - research-system
  - suggestions-pool
  - scheduler
  - dormant
  - augment-skill
source: task-246-251
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

把 IMOSS 从"开发工具"扩展为"开发 + 自我研究成长知识库"，6 步落地 v4 闭环架构（246-251）：

```
信号扫描（251）→ 候选 → 用户审（246 dashboard /suggestions）→ accept 派发
                                                            ↓
                                             scheduler 主控注入（248）
                                                            ↓
                                       augment skill iterate（249 持续追新）
                                                            ↓
                                  蒸馏 + actionable_suggestions（247）
                                                            ↓
                                       建议池显示（246 + 250 queue 视图）
                                                            ↓
                                       用户 accept → task → dev 工作
                                                            ↓
                                       knowledge 沉淀 → 反馈下一轮信号扫描
```

5 条核心铁律贯穿全 v4：

1. **单 worker 串行**（取代 241 的多 tmux 并行设计）：scheduler 注入主控窗口（IMOSS_SESSION='imoss'），不依赖 task-bound Codex 窗口；244 的 task-bound 路径降级为可选 PoC
2. **建议池作为研究产出统一收口**：`.imoss/suggestions/<date>-<slug>.md`，subagent / scheduler / 自动扫**都不直接派发**，所有 task-create / project-init 必须用户在 dashboard 拍 accept
3. **subagent 摘要 → 主 agent 落盘**职责边界：237 的"subagent 不写文件"边界保留；新加的 `actionable_suggestions` 字段也只在 subagent 摘要里返回，由主 agent Write tool 落盘
4. **持续追新**靠"前 N-1 次产出快照 + after:date filter + URL 去重 + iteration/supersedes"四件套（249）；iterate 模式**不能**被 sufficient=true 短路（[[2026-05-07-lesson-iterate-不被-sufficient-短路-task249]]）
5. **dormant 自动化**：30/60/90 天分级（dormant / dormant-cold / archive-suggested）；plan.md frontmatter status 派生 archive-suggested 必须从 iteration state 读，不能只看 plan（[[2026-05-07-lesson-dormant-计算语义边界-task248]]）

## 背景

任务 241 拍板"dev/research 架构整合"路线，原方案是 2-3 课题各自一个 tmux 窗口并行。用户后续修订（v4）改为：单 worker 串行 + 课题池 + 持续追新 + 建议池作为产出收口。原因：

- 用户实际需求是 2-3 个**深度**课题，不是 N 个并发
- 多 tmux 窗口资源开销大且 244 的 task-bound 注入跟 restart-project-window 清 active-session 语义冲突
- 建议池让"研究 → 落地"有显式人工审节点，避免 LLM 噪音填爆 task 池

## 6 个落地步骤

| # | task | 核心改动 |
|---|---|---|
| 1 | 246 | `.imoss/suggestions/` schema + suggestions-fs.ts service + 5 个 tRPC procedure（list/detail/accept/reject/archive）+ dashboard /suggestions 页 + accept modal 多项目 checkbox + slug 路径穿越防护（resolveSuggestionPath） |
| 2 | 247 | augment skill 摘要 schema 加 `actionable_suggestions[]`（type/title/targetProjects/proposedAction/confidence/rationale/body/sourcePaths）+ 主 agent Step 2.5 落盘到 suggestions/（slug 防覆盖 -2/-3/short hash）+ 严格约束 subagent 不直接 task-create |
| 3 | 248 | `tryDirectInjectMainControl` 替代 244 task-bound 主路径 + `computeMinIntervalDays` 动态节奏（queue<3→7d / 3-8→3d / >8→1d）+ `computeDormantStatus` + `applyDormantStatus` 写 plan atomic + 本 tick 内存更新防止刚 dormant 的 task 被本 tick 调度 |
| 4 | 249 | iterate Step 0.5 主 agent 反查 references/external + personal/lessons 拼快照（兼容 inline + multiline `sourceTasks` frontmatter）+ subagent prompt 加持续追新 4 条规则（含"iterate 不被 sufficient 短路"） |
| 5 | 250 | research-fs.ts 加 priority + archive-suggested 派生 + 三级排序（status rank → priority rank → lastIteratedAt asc）+ `research-plan-write.ts` 白名单 wrapper（[[2026-05-07-decision-dashboard-tRPC-写入白名单-wrapper-task250]]）+ Research.tsx 列表项改非交互容器（避免 nested button HTML 错误）+ ⬆⬇ priority 调整按钮 |
| 6 | 251 | research-suggestor.ts 扫信号（blockedTasks: doing>7d / knowledgeGaps: tag<2 lesson + stoplist）+ acceptSuggestionAction 加 'research-topic' 分支强制 IMOSS 派发（servedProjects 仅作上下文）+ Suggestions.tsx 加扫按钮 + accept modal isResearchTopic 特例 |

## 安全 + 工程边界

- **白名单写入口**（250）：dashboard 服务后端不暴露任意 frontmatter 字段写 mutation；只允许 `writeResearchPlanStatusAtomic` / `writeResearchPlanPriorityAtomic` 两个窄口径 wrapper
- **slug 路径穿越防护**（246）：resolveSuggestionPath basename / `.md` 后缀 / resolve-in-dir 校验
- **task-create `--no-session-update`**（246）：dashboard spawn 子进程不污染调用方窗口的任务绑定
- **只读扫描**（251）：file-scanner.scanTasks 扫 done 会 auto-fix frontmatter status；研究主题扫描必须用只读 DI scanActiveTasks（[[2026-05-07-lesson-file-scanner-scantasks-副作用-task251]]）
- **research-topic 派发顺序**（251）：targetProjects 是服务范围非派发目标；acceptSuggestionAction 必须在 generic targetProjects 校验之前分支处理，强制 P001
- **iterate 不被 sufficient 短路**（249）：历史 lesson 让本地命中充分会让追新永远不联网

## 量级与扩展边界

- 适用规模：2-3 个深度长期研究 + 偶发 oneshot 查询
- 不适用：≥10 课题级 / hermes 化 standalone runtime（超出 v4 范围；用户量级不需要）
- queue 满（>8）→ 动态节奏自动加速到每天 1-2 个；priority='high' 跳过 minIntervalDays 立即插队（保留 24h cooldown）
- dormant 自动化让长期失活课题自动降级，避免知识库被无产出课题占满

## 实施验证

- dashboard 测试 184 → 205 case（v4 期间净增 21 case 覆盖关键路径）
- 实测扫信号产生 3 条 onoai task 卡顿候选（卡 14/19/27 天的真实情况）
- 4 commits 关键里程碑：246 `aceb867+52a5a36` / 247 `d3ded85` / 248 `0400963` / 249 `cc237b7` / 250 `168a63c` / 251 `d6ebad8`

## 关联

- [[2026-05-06-decision-imoss-devresearch-架构整合路线-task241]] — v3 原方案
- [[2026-05-07-lesson-iterate-不被-sufficient-短路-task249]]
- [[2026-05-07-lesson-dormant-计算语义边界-task248]]
- [[2026-05-07-decision-dashboard-tRPC-写入白名单-wrapper-task250]]
- [[2026-05-07-lesson-file-scanner-scantasks-副作用-task251]]
- [[2026-05-06-decision-tmux-send-keys-自然语言伪用户输入-task244]] — 主控注入基础设施
- [[2026-05-06-decision-研究执行下沉-subagent-task237]] — subagent 边界

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-decision-codex-worker-接管研究执行-task254|IMOSS 研究执行 worker 从主控 Claude 委派给 Codex 窗口（v5 路径 A）]]
- [[2026-05-07-decision-dashboard-tRPC-写入白名单-wrapper-task250|dashboard tRPC 后端对 plan.md frontmatter 写操作必须经过白名单 wrapper]]
- [[2026-05-07-lesson-dormant-计算语义边界-task248|research dormant 状态计算的 5 条语义边界（无事件不 dormant / latestAttemptAt 而非 lastIteratedAt / done 排除）]]
- [[2026-05-07-lesson-file-scanner-scanTasks-副作用-task251|file-scanner.scanTasks 有副作用（auto-fix frontmatter status），只读场景必须用 DI 替代]]
- [[2026-05-07-lesson-iterate-不被-sufficient-短路-task249|augment skill iterate 模式不能被 imoss_memory_search 的 sufficient=true 短路]]
