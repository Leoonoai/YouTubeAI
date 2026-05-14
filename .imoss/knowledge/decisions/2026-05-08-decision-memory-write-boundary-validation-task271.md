---
id: decision-2026-05-memory-write-boundary-validation
title: memory capture 的分类 / 注入扫描 / 时态校验放在写入边界，不放搜索时
status: active
confidence: 85
tags:
  - inbox
  - capture
  - validation
  - write-boundary
  - prompt-injection
source: task-271
createdAt: 2026-05-08T00:00:00.000Z
relatedDecisions:
  - 2026-05-08-decision-mcp-search-profile-abstain-task270
relatedLessons:
  - 2026-05-08-lesson-agent-memory-write-boundary-needs-validation
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

inbox capture（`scripts/inbox-add.mjs --json-stdin` + `buildCapture()`）写入
frontmatter 之前统一调 `validateInboxInput()`，把以下 4 个维度做成"写入边界"：

1. **classification**（fact/preference/correction/task-summary/journal）启发式分类，
   显式 enum 校验
2. **correction 元字段 all-or-none**：correctsRef + wrongBecause + fix 三选一存在
   则三项必须齐全（缺则 throw）
3. **prompt-injection 启发式扫描**：5 个保守 jailbreak 模式（ignore-instructions /
   system-role-marker / inst-marker / role-override / safety-override），
   patterns 用稳定 id（不复制原始攻击文本）
4. **validFrom/validUntil 时态字段**：非法 ISO date / `validUntil < validFrom` →
   warn-drop，不阻断

**v1 错误处理**：correction 缺字段 + classification 非法 enum → `throw + exit 1`；
其他都 stderr warn 不阻断。

## 背景

mnemos（持久化记忆 MCP）启发：把 correction / 时态有效性 / compaction recovery /
prompt-injection scanning 放在**写入边界**，比放在**读取边界**（search-time）成本
更低、可审计性更高 —— 每条记录在落盘时就带完整元数据，下游所有消费方（triage /
search / dashboard）直接读 frontmatter。

## 理由

1. **早绑定**：写入时算一次 vs 每次 search 都算
2. **可审计**：frontmatter 直接落盘可见，triage 工具自己读
3. **召回率不受影响**：v1 fail-soft 不阻断，召回不变
4. **唯一红线 = correction 完整性**：correction 半结构化等价于"修正失败的修正"，
   比不修正更危险，所以 v1 就阻断

## 操作规范

外部 capture 入口（telegram bot / browser ext / voice）扩展现有 264 schema：
- 显式传 `classification` / `correctsRef+wrongBecause+fix` / `validFrom+validUntil`
- 不显式传 → buildCapture 启发式分类
- prompt-injection 命中：frontmatter 写 `suspectPromptInjection: true + injectionPatterns: [...]`
  (用 stable id，**不复制**原文)，stderr warn，仍写入

## 参考

- spec: `.imoss/rules/core/memory-write-boundary.md`
- 实现: `scripts/lib/inbox-validation.cjs` + `scripts/inbox-add.mjs`
- 264 capture protocol（写入入口）: `.imoss/rules/core/inbox-capture-protocol.md`

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-decision-inbox-capture-first-triage-later-task259|inbox 层用 capture-first / triage-later，4 分支结构化处置]]
- [[2026-05-08-decision-inbox-capture-stateless-wrapper-task264|inbox capture 入口用 stateless CLI wrapper，不开 daemon / 长期 worker]]
- [[2026-05-08-lesson-injection-patterns-stable-id-not-raw-text-task271|prompt-injection 扫描结果用稳定 pattern id，不要写原始攻击文本进 frontmatter]]
