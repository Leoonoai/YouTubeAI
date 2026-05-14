---
title: llm-report-html 集成采用 schema 子集 + React 渲染
type: decision
status: active
confidence: 85
tags:
  - dashboard
  - research
  - report
  - schema
  - react
source: task-311
audience: work
createdAt: 2026-05-10T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# llm-report-html 集成采用 schema 子集 + React 渲染

## 决策

IMOSS 研究系统集成 `yansircc/llm-report-html` 时，不安装 Go 工具链，也不直接跑源项目 CLI。v1 只沿用源项目 schema 的静态报告子集，由 dashboard 现有 React/Vite/Tailwind 栈渲染 JSON 报告。

报告作为研究 iteration 的 add-on：`references/external` 和 `knowledge/personal` 这条 markdown 蒸馏链路保持主产物地位，HTML 报告只提供 dashboard 内的可读视图。

## 背景

任务 311 的目标是让研究系统每次 iteration 完成后额外产出结构化 JSON 报告，并在 dashboard `/research` 详情页提供“查看研究报告”入口。调研记录确认源项目本质是 Go CLI + Claude Code skill，`schema.json` 是单一事实源，AI agent 写 JSON，CLI render 成自包含 HTML。

任务调研还确认源 template 是 vanilla JS，不绑定 React/Vue；dashboard 已经是 Vite + React + Tailwind 栈，新增 ReportViewer 不需要引入新构建工具。

## 理由

v1 研究报告是静态文档，主要承载叙事、数据和来源引用，所以不需要 source schema 里的 reactive 能力。plan 明确把 `state / computed / input / if / $bind` 推出 v1。

保留 `heading / paragraph / quote / code / hr / callout / list / table / timeline / definition / faq / stat / details / aside` 这 14 个 section type，足够覆盖研究报告的主要展示形态。`image / mermaid / diagram / tabs / columns` 被列为非目标或 v2 方向，避免第一版复杂度过高。

React 渲染器与 dashboard 当前技术栈一致，避免为了单个报告功能引入 Go build/runtime 路径。progress 记录显示最终落地了 shared TS 类型、server zod schema、report fs service、ReportViewer、research report 路由和 worker 集成，并通过真实报告验证。

## 操作规范

新增报告能力时保持三条边界：

1. JSON schema 和 dashboard 渲染器优先兼容源项目字段名和值约束，但只实现 IMOSS v1 子集。
2. client 不 runtime import server zod schema；纯 TS 类型放 shared，server 校验留在 service 层。
3. 报告文件落到 `.imoss/reports/research/<taskId>/<reportId>.json`，`reportId` 使用 URL-safe 时间戳，JSON 内保留真实 ISO 时间。

研究 worker 侧不直接“运行 skill”。task 311 最终采用 `codex-dispatch --mode research-report --wait`，由 dispatcher 生成 prompt，Codex worker 按 `.claude/skills/imoss-research-report/SKILL.md` 契约写报告和 `research_report_done` 事件。

## 参考

- 来源任务：task 311
- 关键文件：`.imoss/tasks/done/311-集成-llm-report-html研究系统产-html-报/plan.md`
- 进度证据：`progress.md` 记录 `npm test 285 全过`、fixture 和真实 task 252 浏览器 e2e、两个 commit 已 push
