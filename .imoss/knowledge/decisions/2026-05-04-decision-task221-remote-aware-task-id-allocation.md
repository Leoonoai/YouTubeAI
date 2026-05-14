---
title: task-create 分配 ID 前合并本地与 origin/main 最大任务号
type: decision
status: active
confidence: 88
tags:
  - tasks
  - git
  - workflow
  - id-allocation
source:
  taskId: '221'
  taskDir: .imoss/tasks/done/221-task-create-分配-id-前-fetch-orig
sourceTasks:
  - '221'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`scripts/task-create.mjs` 分配新任务 ID 前先 best-effort `git fetch origin main --quiet`，然后用本地 active/done 任务目录和 `origin/main` 上 `.imoss/tasks/(active|done)/NNN-*` 路径中的最大 ID 共同计算 `nextId = max + 1`。

## 理由

多机器并发时，如果两台机器都只看本地最大任务号，就会创建相同 ID。后 push 的机器即使 rebase，也会留下任务号撞车，需要人工重编。分配前读取远端任务树可以在大多数联网场景下提前避开冲突。

## 关键规则

- fetch 失败不阻塞 task-create：先尝试使用已缓存的 `origin/main`，没有 cached ref 才退回纯本地。
- 远端解析不能看行首，因为 `git ls-tree -r --name-only` 返回的是文件路径；必须从 `.imoss/tasks/active/NNN-slug/...` 或 `.imoss/tasks/done/NNN-slug/...` 的路径段提取 ID。
- `--id` 显式指定时保持 override，不触发自动分配逻辑。
- 逻辑抽到 `scripts/lib/task-id-allocation.mjs`，用临时 repo/bare origin 测试，不改真实 remote。

## 证据

任务 221 新增 task-id-allocation 单测并通过 13 个用例；CLI dry-run 验证无 `--id` 时取到远端感知的新 ID，显式 `--id` 不触发 fetch。
