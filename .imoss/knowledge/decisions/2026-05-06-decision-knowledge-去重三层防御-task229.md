---
id: decision-2026-05-knowledge-去重三层防御
title: >-
  knowledge corpus 去重做三层（MCP find_similar 工具 + audit 脚本 + distill skill 蒸馏前查），用
  Jaccard 双指标算文档相似度而非 BM25
status: active
confidence: 85
tags:
  - knowledge
  - dedup
  - mcp
  - distill
  - jaccard
  - similarity
source: task-229
createdAt: 2026-05-06T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

给 IMOSS knowledge corpus 加 3 层去重防御，互相补：

1. **MCP `imoss_memory_find_similar`**（在线查询入口）
   - 输入：一段文字 / `{title, body}` 草稿对象 / 现有条目 id
   - 输出：corpus 里相似度 ≥ threshold（默认 0.4）的条目数组
   - 算法：**Jaccard 双指标**（title token Jaccard × 0.6 + body token Jaccard × 0.4）
   - 输入只有 body 时退化为 body-only 评分（避免被 title weight 0.4 压低分）

2. **`scripts/knowledge-dedup-audit.mjs`**（离线扫全 corpus）
   - 一次性扫所有 corpus 条目两两计算相似度
   - 模式：默认 stderr 报告人读 / `--check` 退出码 1 让 CI 拦 / `--json` 结构化输出
   - 用来定期清理已混入的重复条目

3. **`distill-external` skill 蒸馏前 step**（写入端拦）
   - 蒸馏前先调 `imoss_memory_find_similar({title, body})`
   - 命中相似 ≥ threshold → confidence 自降、status 强制 'draft'、body 加 `## 与现有条目的关系`节列 `[[<slug>]]`
   - 不直接拒绝写入：用户/agent 看到 draft 后人工裁决

## 背景

225 → 230 期间反复出现一个问题：用 imoss-query-augment 跑同一个主题第二次/第三次时，生成的 personal lesson 跟已有条目大量重叠，但 augment 的 fetch+distill 没检测，每次都生新文件。corpus 越来越像"同一主题在不同文件名下重复"——搜索时被多份重复条目挤掉真正多样化的 hit。

## 理由

**为什么用 Jaccard 而不是 BM25/RRF / 向量嵌入**：

- BM25 / RRF 算的是 **query → body 匹配度**，是单向问题。文档去重是 **body ↔ body 双向相似**，需要不同算法
- 向量嵌入（cosine similarity）效果好但需要 LLM SDK + 计算开销 + 模型选择问题；IMOSS 当前坚持"hooks 层无 LLM SDK 硬依赖"
- Jaccard token 重叠简单、可解释、无依赖；阈值 0.4 实战够分辨"显著重叠"vs"恰好同主题不同角度"

**为什么 title × 0.6 + body × 0.4**：

- title 是高浓缩信号，标题 Jaccard 高大概率真重复
- body 长容易因为通用词 token 多重叠拉高分；权重压一下
- 纯 text 输入（无 title）退化 body-only 是兼容措施，避免误以为"没标题"=低分

**为什么 distill skill 不直接拒绝**：

- 自动判定可能误伤——0.5 临界值的 hit 实际可能是不同视角值得保留
- 改成 confidence 降级 + draft 状态 + body 自动加关联节，让用户/agent 后续筛
- 严格拒绝路径留给阈值 ≥ 0.7 的明显复制

## 操作规范

**写新 lesson/decision 时**：
- 自动通过 distill-external skill 蒸馏 → 已自动调 find_similar
- 手写时若担心重复：`node -e "..."` 调 MCP 或在 dashboard /knowledge 搜
- 手写产物落盘后跑 `node scripts/knowledge-dedup-audit.mjs` 看有没有意外重叠

**周期性维护**：
- 每月或每 100 条新 knowledge 时，跑 `knowledge-dedup-audit.mjs` 看 stderr 报告
- 真重复 pair 手动合并 / 删冗余条目

**阈值调整**：
- 默认 0.4 是合理起点
- 如果 audit 误报多：调到 0.5
- 如果蒸馏漏检多：保持 0.4 不变，加 `--threshold` 参数让 CLI 调用方决定

## 不做

- **不**做向量嵌入 dedup（IMOSS 无 SDK 依赖原则）
- **不**自动删除疑似重复条目（数据保留优先于自动清理）
- **不**在 search 端去重（去重该在写入端做，搜索端是只读）

## 验证

- task 225 两份 dspy.ai 同源原文蒸馏 → find_similar 相似度高 → 第二份标 draft + body 加关联节
- audit 扫现有 corpus → 报 0-3 个真实近似 pair（不误判 4 条 DSPy lesson 为重复——它们确实内容不一致）

## 参考

- task 229 optional/mcp/server.mjs `imoss_memory_find_similar`
- task 229 scripts/knowledge-dedup-audit.mjs
- task 229 .imoss/skills/distill-external.json prompt 4.2 节
- [[2026-05-06-lesson-mcp-coverage-语义启发式-task228]]（同一系列质量加固）
