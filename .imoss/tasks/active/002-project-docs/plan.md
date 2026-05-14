---
id: "002"
title: "项目资料中心"
priority: P1
status: living
neverArchive: true
waitingReason: ""
assignedAgent: ""
blockedBy: ["001"]
blocks: []
mode: standard
createdAt: 2026-05-14T07:27:14.530Z
updatedAt: 2026-05-14T07:27:14.530Z
---

# TASK-002: 项目资料中心

> ⚠️ **永不归档**（IMOSS 约定 / 132 统一）：本任务是 **YouTubeAI 的长期资料中心**，`status: living`，`neverArchive: true`，永远不进 `done/`。归档脚本 `scripts/task-archive.mjs` 会拒绝把它移出 `active/`。

## 目标

建立 **YouTubeAI** 的长期资料中心：集中存放所有外部输入参考资料、项目级文档、跨任务可复用的素材。作为所有新任务的"输入材料入口"和"产出总结的回流点"。

## 组件

### `.imoss/references/` — 外部原文集中区

- 存放用户提供的**外部原文**（视频文案转写 / 文章 / 书籍摘录 / 对谈实录 / 调研输入等），格式不限
- `README.md` 作为索引，按日期倒序列出所有 reference，含作者/主题/类型/触发任务/吸收 lesson
- **命名约定**：`YYYY-MM-DD-<作者或主题>-<具体主题>.md`
- **原文保留**：不要修改或美化原文（包括语音识别错漏），保持原样；结构化吸收放 `.imoss/knowledge/lessons/`
- **任务里不复制原文**：新任务要分析某份原文 → 在 plan.md 里引用 `.imoss/references/YYYY-MM-DD-...md` 路径，**不**在 task dir 放 source-article.md
- 规则全文和索引表见 `.imoss/references/README.md`

### `.imoss/knowledge/` vs `.imoss/references/` 的分工

- `references/` 装**原文材料**（别人写的、用户提供的）
- `knowledge/` 装 **distilled lesson/decision**（我们自己总结的经验）
- 两者**不合并**。原文不美化，总结不掺原话

## 使用工作流

1. **任务开始时**：用户提供外部原文 → 直接写入 `.imoss/references/YYYY-MM-DD-*.md`，同步更新 `references/README.md` 索引
2. **任务执行时**：plan.md 里用路径引用 references 文件，不复制
3. **任务收尾时**：任务产出里有"值得长期参考的部分"（不是 distilled lesson，而是原始素材）→ 可以回流到 references，同步索引
4. **知识沉淀**：distilled lesson 仍然走 `.imoss/knowledge/lessons/`，不混淆

## 持续待办（长期滚动，不是 checklist）

- 新增 reference 时更新索引表
- 历史任务里散落的 source-article.md **不主动迁移**，保持原位；在 references 索引里用"历史引用"链接指过去即可

## 完成标准

本任务**没有终点**（living）。运行状态可以在"有新 reference 要加"和"空闲"之间滚动，但永远不转 `done`。

## 非目标

- **不**把 `knowledge/lessons/` 和 `references/` 合并
- **不**为 references 建自动索引/爬虫/search（手工维护 README.md 表格即可）
- **不**追溯迁移历史任务里的 source-article.md（保持原位 + 索引链接即可）
- **不**在 task dir 里复制 references 内容（严格用路径引用）
