---
id: decision-2026-05-研究执行下沉-subagent
title: 研究课题的 search→fetch→distill 循环下沉到 Agent 工具的 subagent 执行，让多课题 context 天然隔离
status: active
confidence: 85
tags:
  - subagent
  - agent-tool
  - context-isolation
  - research-system
  - claude-code
source: task-237
createdAt: 2026-05-06T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

把研究课题（imoss-query-augment 的核心循环）从主 Claude Code agent 下沉到 **Agent 工具**（subagent_type=general-purpose）执行。主 window 只编排：建 task → spawn subagent → 收摘要 → 写 plan → 归档/状态推进。subagent 在独立 context 里跑 search → fetch → distill → re-search 循环，跑完返回结构化摘要。

并行多课题：主 window 同一轮回复内并行发出多个 Agent calls，每个 subagent context 完全隔离。

3 个触发路径共用同一 skill：
- 手动入口（用户说"研究 X"）：mode=oneshot，建 task + spawn subagent + 归档
- 232 auto-research（imoss-query-augment 用户层入口）：mode=oneshot 同上
- 234 cron iterate（hook 注入"迭代 NNN"）：mode=iterate，复用 task NNN，写 lastIteratedAt + 删 marker，**不**新建/归档

## 背景

231 → 234 把研究系统的数据/调度建好，但执行依然在主 window 跑：用户问 "研究 A"（subagent 内部跑 fetch+distill）→ 同 window 紧接着用户问 "研究 B"，B 复用了 A 留下的 context（拉 fetch 时把 A 主题词带进了 query），多课题间相互污染。task 237 调研后用户拍板方案 B：subagent 下沉。

## 理由

1. **Claude Code Agent 工具天然隔离 context**：每个 spawn 的 subagent 拿到独立 chat history，看不到主 window 当前对话内容，prompt 里写什么就只跑什么
2. **主 window 上下文只承载编排逻辑 + 摘要**：subagent 摘要进 plan.md，主 window 只引用关键路径不复述原文，主 window 长期工作的 context 占用几乎线性
3. **多 query 并行**：单条消息内可发 N 个 Agent call，瓶颈在网络（fetch/WebSearch）而非串行 LLM 调用
4. **mode=oneshot vs iterate 共用 skill**：避免两条 skill 平行漂移；用 prompt 内 mode 字段区分行为分支
5. **subagent 不能递归调本 skill**：spawn 新 Agent 会无限递归。subagent 必须直接执行 search/fetch/distill 子操作（用 MCP 工具 + WebSearch + WebFetch + Bash + Write），不调 imoss-query-augment skill 自己

## 操作规范

**主 agent 编排（imoss-query-augment.json prompt）**：
- Step 0：解析 query + 建 task（mode=oneshot 时；iterate 跳过）
- Step 1：spawn subagent（subagent_type=general-purpose，prompt 用模板填好 placeholders）
- Step 2：收摘要写 plan.md（## 研究问题 / 已抓资料 / 已蒸馏结论 / 迭代历史 / 完成标准）
- Step 3：归档（oneshot）或 写 state 删 marker（iterate）+ 244 后 writeEvent
- Step 4：基于摘要给用户最终答案

**subagent 工具白名单**：
- MCP: imoss_memory_search, imoss_memory_find_similar
- Web: WebSearch, WebFetch
- Bash: yt-dlp / knowledge-link-check.mjs
- 文件: Read, Write

**subagent 严格约束**：
- 不递归调本 skill（无限递归）
- 每个 query 最多一次 fetch+distill 循环（不为凑命中反复抓）
- 失败链可见（fetch 0 / distill 0 / 工具失败都进 errors[]）
- 不在本地命中充分时强抓（sufficient=true 直接答）
- 自动产物只进 personal/（不污染 decisions/lessons/）

**主 agent 严格约束**：
- 每 query 至多 1 个 subagent（subagent 内部 cycle 失败也不重 spawn）
- subagent 摘要不可信任时不重试，照实写 plan + 告知用户失败
- 不替 subagent 做 fetch/distill（主 agent 不应在 subagent 之外触发 WebFetch / Write 写 references/personal）
- 不污染主 agent context（subagent 摘要进 plan.md，主 agent 只引用关键路径）

## 风险与不做

- **不实现 subagent ↔ subagent 直接通信**：subagent 间互相隔离是 feature 不是 bug；要协作时通过 fs 落盘交换
- **不指定 subagent model**：默认继承父级（Opus）保证质量；后续观察 cost 再考虑降 sonnet
- **失败兜底不重试**：subagent 报告"0 抓取 / 0 蒸馏 / 工具失败"时主 agent 不重试，照实记 plan.md + Telegram 通知用户
- **distill 不独立下沉**：与 fetch 共一个 subagent；fetch 与 distill 间需共享 references 路径列表，subagent 间不能传状态

## 参考

- task 237 .imoss/skills/imoss-query-augment.json prompt 完整改写
- task 232 (oneshot 自动建/归档) + task 234 (cron iterate 入口) 共用此 skill
- [[2026-05-06-decision-imoss-devresearch-架构整合路线-task241]]（241 在此基础上加跨对话长期推进）
- Claude Code Agent 工具文档（subagent_type / model 继承机制）

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-decision-codex-worker-接管研究执行-task254|IMOSS 研究执行 worker 从主控 Claude 委派给 Codex 窗口（v5 路径 A）]]
- [[2026-05-07-decision-v4-pkm-闭环架构-task246-251|IMOSS 自我研究成长系统 v4 架构 — 单 worker 串行 + 建议池统一收口 + 持续追新闭环]]
