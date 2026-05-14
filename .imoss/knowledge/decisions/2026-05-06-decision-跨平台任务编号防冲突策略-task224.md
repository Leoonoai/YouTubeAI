---
id: decision-2026-05-跨平台任务编号防冲突策略
title: 多机/多平台并行操作 IMOSS 时按"原子工作单元"提交+推送，不积攒、不跨任务批量
status: active
confidence: 85
tags:
  - git
  - task-id
  - workflow
  - multi-machine
  - conflict
source: task-224
createdAt: 2026-05-06T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

任务编号是 IMOSS 全局递增的，由 `task-create.mjs` 扫 `.imoss/tasks/{active,done}/` 找最大值 + 1 决定。**多机或多平台（WSL + Mac、IMOSS + 子项目仓库）并行操作时，编号生成依赖于 git 同步状态**。

防冲突策略（按"原子工作单元"提交+推送）：

1. **task-create 后立刻 commit + push**：`scripts/task-create.mjs` 写完文件后**不要**等到工作完成才 commit；马上 `git add` task 目录 + BACKLOG.md，commit `chore(NNN): 创建任务`，并立即 `git push`。这一步是廉价的（几十 KB），但锁定了编号。
2. **跨设备/跨终端切换前同步**：换 Mac / 换 tmux 窗口 / 进新 IMOSS session 之前，**先 `git pull --rebase`**，让本地 task 编号视图最新。
3. **绝不积攒"待 push"的 task-create commit**：积攒会让另一台机器/平台分配同一个编号。
4. **`commit + push` 是一个原子动作**：任何"我等会再 push" 的犹豫都是 224 教训的源头。

## 背景

历史上出现过 task 编号冲突——两台机器（WSL + Mac）几乎同时建任务，git pull 没及时同步，task-create 各自分配了同一个 NNN。事后合并时一个任务只能改名，BACKLOG / handoff / context 全要回溯。

## 理由

- **task-create 拿编号时唯一可信源是 fs 状态**：脚本不可能读 origin 远端最新，必然依赖本地 fs。本地 fs 不及时同步 = 编号不新鲜。
- **commit 自动加锁**：编号一旦 commit + push，远端就有了证据。后来者 pull 时会看到，避开。
- **任务文件改动小**：task-create 第一刻只产 5 个文件 + 几百行模板，commit + push 网络开销可忽略。**没有"为了不打断 flow 暂不 commit"的理由**。

## 操作规范

**写代码 / 调研中**：可以批量 commit（攒到一个 logical 节点 commit）。
**`task-create.mjs` 落盘的瞬间**：必须立刻：
```bash
node scripts/task-create.mjs "新任务 X" --project IMOSS
git add .imoss/tasks/active/<NNN>-*/  .imoss/tasks/BACKLOG.md
git commit -m "chore(NNN): 创建任务"
git push origin main
# 然后才开始读源码 / 写 plan
```

跨设备切换前：
```bash
git fetch && git pull --rebase
ls .imoss/tasks/active/  # 确认看到最新编号
```

子项目仓库（独立 repo）：同一规则适用，按子项目自己的编号空间。子项目之间编号互不冲突（不同 repo），不需要跨 repo 同步。

**不做**：
- 不要 `git push --force` main（永远；多机协作下 force push 是数据丢失保证）
- 不要积攒 task-create + 实施 commit 一次性 push（积攒越久编号风险越大）
- 不要在两台机器开同一个 IMOSS terminal session 操作（`IMOSS_SESSION` 串了 fallback global 会乱）

## 风险与边界

- **网络断开时**：暂不能 push 但可以 commit（locked-on-local），恢复连接后第一时间 push。期间不要在另一台机器做 task-create
- **MQ-style "工作再批量推"工作流**：不适用 IMOSS（编号是 stateful 的强一致需求）
- **CI 自动建任务**：如果将来引入 CI 触发 task-create，必须在 CI 内完整 commit+push，不能放到 worker 队列异步推

## 参考

- task 224 plan.md 完整决策推导
- `scripts/task-create.mjs` 编号生成逻辑（扫 fs 找 max+1）
- 历史冲突 commit log（人工合并的痕迹）
