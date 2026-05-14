---
id: decision-2026-05-knowledge-class-memory-vs-wiki
title: knowledgeClass 二分检索面 — memory（个人记忆）vs wiki（事实知识）
status: active
confidence: 85
tags:
  - knowledge
  - retrieval
  - class
  - memory
  - wiki
source: task-261
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

加 `knowledgeClass` frontmatter 字段，二分：

- **`memory`** — 个人记忆类（user preferences / 个人决策 / 经验 lesson）
- **`wiki`** — 事实知识类（reference / 项目文档 / 技术 spec）

MCP `imoss_memory_search/list` 接受 `knowledgeClass` 参数；Dashboard `/knowledge`
默认 `wiki`，切 Memory 看 memory source / All 看全部。

## 理由

iter2 supersedes iter1 的"profile-scoped 是必要不充分" — 仅靠 profile 不够，
还需要区分**检索面性质**：
- memory 类 = 个人化、上下文依赖、不应被一视同仁与 wiki 混排
- wiki 类 = 事实参考、无个人色彩、可被任何 agent 引用

class 区分让 dashboard 浏览 + agent 注入策略更清晰：
- dashboard /knowledge：默认 wiki（避免个人记忆混入工具浏览）
- agent system prompt 注入：可分别控（memory 配合 270 abstain / wiki 当事实参考）

## 操作规范

- derived default **强**：`source=memory` → `memory` / `source=knowledge|reference` → `wiki`
- 几乎所有 corpus 自动派生，**用户极少需要显式标注**
- MCP **不**做 profile→class 隐式映射（向后兼容）；调用方明确传 `knowledgeClass` 才生效

## 参考

- spec: `.imoss/rules/core/knowledge-classes.md`
- 实现: `optional/mcp/memory-corpus.mjs:resolveKnowledgeClass`
