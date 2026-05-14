---
title: lint 范围必须显式 scope，不要一次性收紧全 corpus
type: lesson
status: active
confidence: 85
tags:
  - lint
  - scope
  - schema
  - gradual-rollout
source: task-263
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# lint 范围必须显式 scope，不要一次性收紧全 corpus

## 教训

新加 lint 规则上线时，**必须显式限定扫描范围**（哪些目录、哪些文件类型），
不要默认扫整个 corpus。否则 200+ legacy 文件全部 warning，淹没真问题。

## 原因

263 personal-knowledge-lint 第一版差点设计成"扫 .imoss/knowledge/**"。
review 阶段 Codex R1 拍：
- IMOSS 已有 200+ knowledge md，没人按新 schema 写过
- 一次扫全部会输出 100+ 行 warning，用户看一眼直接关掉，lint 变废
- 收紧 schema 应该**按文件夹分批**，让每批新建文件按新规则写，旧文件等空闲时迁移

## 后果（如果忽略）

- 用户第一次跑 lint 看到几百行 warning，直接 disable lint（再也不开）
- pre-commit hook 频繁 warn，开发者养成"忽略 lint"的习惯
- 真正的 schema 红线（sensitivity 泄漏）淹没在 noise 里

## 正确做法

```js
// scripts/personal-knowledge-lint.mjs
const SCAN_PATHS = [
  '.imoss/knowledge/personal',
  '.imoss/references/external',
  // 暂不扫 lessons/decisions/solutions/inbox/tasks
];
```

按这个顺序收紧：
1. **第 1 批**：新写文件高密度的目录（personal / external）
2. **第 2 批**（如果第 1 批稳定）：扩到 lessons / decisions
3. **第 3 批**（如果有信号）：扩到 inbox / tasks

每批的"成功"标准：开 lint 后 warning 数 < 5（不淹没用户）。

通用规则：
- **lint 上线 = 渐进推广，不是一次性合规**
- pre-commit 用 warn（不 block），CI 用 strict（任一 error block）
- 提供 `--rule <name>` 让用户单独跑某条规则，便于 debug

## 参考

- 实现: `scripts/personal-knowledge-lint.mjs:SCAN_PATHS`
- 相关：[[2026-05-08-decision-personal-knowledge-lint-5-rules-task263]]
