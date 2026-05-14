---
title: sync-rebuild 必须 idempotent — 跑 N 次结果与跑 1 次相同
type: lesson
status: active
confidence: 85
tags:
  - sync-rebuild
  - idempotent
  - invariant
  - cache
source: task-266
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# sync-rebuild 必须 idempotent — 跑 N 次结果与跑 1 次相同

## 教训

任何"从 markdown 派生 SQLite/index/cache"的脚本（如 `sync-rebuild.mjs`），
必须 **idempotent**：连跑两次或更多次，输出表 count + 抽样行内容必须完全一致。

## 原因

非 idempotent sync-rebuild 会导致：
- 第一次跑后 dashboard 显示 174 条 knowledge
- 第二次跑后变 175 / 173 / 重复行 / 排序变化
- 用户改 markdown → 跑 sync → 看到结果 → 又被另一次 sync 改掉

测试 idempotent 的最小方法：
```bash
node scripts/sync-rebuild.mjs
sqlite3 dashboard/data/imoss.db 'SELECT COUNT(*) FROM knowledge;' > /tmp/c1
sqlite3 dashboard/data/imoss.db 'SELECT id FROM knowledge ORDER BY id LIMIT 20;' > /tmp/s1

node scripts/sync-rebuild.mjs
sqlite3 dashboard/data/imoss.db 'SELECT COUNT(*) FROM knowledge;' > /tmp/c2
sqlite3 dashboard/data/imoss.db 'SELECT id FROM knowledge ORDER BY id LIMIT 20;' > /tmp/s2

diff /tmp/c1 /tmp/c2 && diff /tmp/s1 /tmp/s2 && echo "idempotent ✓"
```

## 后果（如果忽略）

- 用户不知道"真相"是 markdown 还是 SQLite
- dashboard 数据每次 refresh 看着像变（实际是非 idempotent 噪声）
- 调试时无法 reproduce

## 正确做法

实现 sync-rebuild 时遵循 derived-cache 模式：
1. 第一步：`DELETE FROM knowledge`（先清空，不是 INSERT OR UPDATE）
2. 第二步：扫所有 markdown 文件
3. 第三步：批量 INSERT
4. 第四步：commit

避免：
- 增量 update（容易留下旧行）
- 基于时间戳的 dirty check（mtime 不可靠）
- 在 sync 过程中改 markdown（race condition）

CI / pre-push hook 可加 idempotent 验证：跑两次 + diff，不一致 fail。

## 参考

- 实现: `scripts/sync-rebuild.mjs`
- 验证 fixture: 266 任务实施时跑过 count=174 一致
- 相关：[[2026-05-08-decision-markdown-source-of-truth-task266]]
