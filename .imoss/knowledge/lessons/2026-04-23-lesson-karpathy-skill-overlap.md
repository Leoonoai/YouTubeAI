---
title: 评估外部 Claude Code skill 前先做 overlap 映射
type: lesson
status: active
confidence: 85
tags:
  - skill-evaluation
  - rules
  - engineering-behaviors
  - external-plugin
source:
  taskId: '196'
  referenceDoc: .imoss/references/2026-04-23-forrestchang-karpathy-llm-coding-pitfalls.md
relatedKnowledge:
  - type: lesson
    slug: 2026-04-23-lesson-llm-wiki-gbrain-overlap
createdAt: 2026-04-23T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Lesson: 评估外部 Claude Code skill 前先做 overlap 映射

## 情境

用户抛一个外部 skill / plugin repo（例如 `multica-ai/andrej-karpathy-skills`），希望"吸收进 IMOSS"。第一反应容易是"好主意，装上看看"或"复制一份到 `.imoss/rules/`"——这两种都会产生**重复规则**。

## 正确动作

先做 **overlap 映射**，再决定吸收方式：

1. 把外部 skill 的每条核心规则 / 条款列出来（Karpathy 4 / Addy 6 / 类似的扁平化准则集）
2. 对 IMOSS 现有的 `rules/core/*.md` + 根 `CLAUDE.md` 铁律 + memory feedback 做覆盖度表
3. 按覆盖度决定：
   - **完全覆盖（≥ 95%）**：只归档 references + 写 lesson 说明已评估过；**不动规则本体**
   - **部分覆盖（50-95%）**：提取**不覆盖的增量点**作小任务；只改那一处
   - **大部分未覆盖（< 50%）**：开中/大任务考虑引入新 rule 文件或新 skill

## 关键洞察

- **规则重复比规则缺失危害更大**：两份近乎相同的规则会在未来某次修订时漂移，agent 读到矛盾规则会偏向"先读到的那份"，执行不稳定
- **外部 skill 的价值不是条款本身，而是它们的组合与表达**：Karpathy 用"loop 验证"把"goal-driven"做成可执行动作，这种表达形式是增量，哪怕条款本身已覆盖
- **CLAUDE.md plugin 安装是最诱人的陷阱**：零迁移成本 → "先装着看"→ 两套并行规则永不合并

## 决策分支图

```
外部 skill 来了
    │
    ▼
做 overlap 映射（列条款 × IMOSS 规则对照表）
    │
    ├─ 完全覆盖 → 归档 references + 写 lesson（本任务路径）
    │
    ├─ 增量点 ≤ 1 条 → 补 1-2 行到现有 rule，不开新 skill
    │
    ├─ 增量点 2-5 条且同主题 → small 任务补进现有 rule 文件
    │
    └─ 增量点 ≥ 5 条或需要工具化 → 开 medium 任务，评估是否写新 rule / skill
```

## 反例（要避免的做法）

- ❌ "装上 plugin 让它和 IMOSS 规则共存"——未来每次规则修订都要同步两边
- ❌ "直接把 Karpathy CLAUDE.md 合并到根 CLAUDE.md 末尾"——根 CLAUDE.md 已经够长，叠加会稀释关键铁律
- ❌ "复制 python 反例到 118 engineering-behaviors"——IMOSS 是 TS 为主，python 例不贴合，还要额外维护两种语言

## 对 IMOSS 的 actionable 建议

- 未来用户再带来外部 skill / CLAUDE.md 时，**先读 `.imoss/rules/core/engineering-behaviors.md` 和根 `CLAUDE.md`**，再做覆盖度表
- 如果本 lesson 见效，可以考虑把"overlap 映射"做成一个 `evaluate-external-skill` skill（但目前样本量 1，尚不够，只记为可选）

## 关联

- 触发任务：196
- 原文存档：`.imoss/references/2026-04-23-forrestchang-karpathy-llm-coding-pitfalls.md`
- IMOSS 对比基线：`.imoss/rules/core/engineering-behaviors.md`（118 号任务产出，自身吸收自 addyosmani/agent-skills）
- 下一次应用：[[2026-04-23-lesson-llm-wiki-gbrain-overlap|197 lesson 套用本决策分支图做 llm-wiki + gbrain 覆盖度评估]]

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-04-23-lesson-llm-wiki-gbrain-overlap|LLM Wiki + GBrain 外部知识库系统对 IMOSS 的 overlap 映射结论]]
