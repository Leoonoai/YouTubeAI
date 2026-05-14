---
id: lesson-2026-04-09-subproject-hook-gap
title: 109 post-commit 审计未覆盖子项目 git——onoai 实锤脱节案
status: active
confidence: 85
tags:
  - task-workflow
  - git-hooks
  - sub-project
  - plan-workflow
createdAt: 2026-04-09T00:00:00.000Z
sourceTasks:
  - '109'
relatedLessons:
  - 2026-04-task-misplacement-and-state-drift
fixedIn:
  - commit: 3ebac51
    date: 2026-04-09T00:00:00.000Z
    summary: >-
      install-git-hooks 遍历 registry 装子项目；post-commit 向上搜 IMOSS 根；移除吞掉告警的
      2>/dev/null
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 事件

2026-04-09 下午用户让 agent 梳理 onoai 下 18 个 active 任务为什么未完成。agent 第一遍读 context.json 得出"11 个卡在 investigating"的结论，准备去清理 014~018 的空壳。

在动手前交叉核对 `git log --all | grep "feat(NNN)"`，发现真相：

| 任务 | context.json state | 真实情况 |
|---|---|---|
| 005 PRD 审查 | investigating | 2 commits 含"执行完成" |
| 006 询价邮件 | investigating | feat 已 merged |
| 007 Escrow | investigating | feat 已 merged |
| 008 后台中文 | investigating | feat 已 merged |
| 011 侧边栏重构 | investigating | plan 详列 4 块已完成 |
| 012 UI 修复 | investigating | fix 已 merged |
| 014 Task A | investigating | feat 已 merged |
| 015 Task B | investigating | feat 已 merged |
| 018 Task E | investigating | feat 已 merged |
| 019 TLD 增强 | planning | 7 commits, 所有 criteria [x] |

11 个任务里 10 个已经"做完了但没关单"。plan.md 永远是 TODO 模板，BACKLOG.md 进行中/待办两个节还出现了 ID 冲突（002~006 既在进行中又在待办，任务名完全不同）。

## 与 lesson-2026-04-task-misplacement-and-state-drift 的区别

前一份 lesson（108/109）已经识别"plan-code drift"这类反模式，并在 IMOSS 仓库里装了 `scripts/git-hooks/post-commit` + `scripts/hooks/post-commit-audit.mjs`。那个 hook 的作用是：解析 `feat(NNN):` commit，回查对应任务的 plan 状态，非阻塞告警。

**但 onoai 的 commit 进来时没有任何 hook 告警。** 原因是：

```bash
$ ls /home/devuser/dev/imoss/.git/hooks/post-commit         # ✅ 存在
$ ls /home/devuser/dev/imoss/projects/onoai/.git/hooks/post-commit  # ❌ 不存在
```

**onoai 是独立 git 仓库**，它自己的 `.git/hooks/` 目录没有被 `scripts/install-git-hooks.mjs` 触及过，所以在 onoai 里 commit `feat(014): Task A ...` 的时候，post-commit-audit 根本没运行。

## 根因

`scripts/install-git-hooks.mjs` 只往 IMOSS 根的 `.git/hooks/` 写，没有遍历 `projects/*/` 下面的子仓库。`scripts/setup.mjs` 在集成这一步时也没处理多仓库。

设计时的隐含假设是"commit 都发生在 IMOSS 仓库内"，忽略了 `projects/<name>/` 下面每个都是独立 `.git`，子项目实际开发的 commit 都在各自仓库里——也就是 109 最应该保护的区域反而完全没被覆盖。

## 修复（commit 3ebac51，2026-04-09）

1. **`scripts/install-git-hooks.mjs` 增加 `MULTI_REPO_HOOKS` 集合**：目前只含 `post-commit`。安装时遍历 `.imoss/registry.json`，对每个注册子项目的 `.git/hooks/` 都装一份。post-merge / pre-commit 是 IMOSS 专用（依赖 `dashboard/` 目录），不会误装到子项目
2. **`scripts/git-hooks/post-commit` 向上搜索 IMOSS 根**：最多走 4 级找含 `scripts/hooks/post-commit-audit.mjs` + `.imoss/registry.json` 的目录。同一份 hook 文件可在 IMOSS 或任何子项目执行，定位逻辑纯 shell 不依赖环境变量或硬编码路径
3. **移除 hook 内的 `2>/dev/null`**：原脚本把 audit 脚本的 stderr 一并吞了，audit 的黄色告警从来没真正到达过用户终端。audit 本身 defensive exit 0，不会产生噪声
4. **端到端验证通过**：在 onoai 里 `git commit --allow-empty -m "feat(999): test"` 正确触发"任务 999 找不到"告警；`feat(013)` 触发 review-log 轨迹断链告警

## 剩余限制（follow-up）

**audit 脚本的任务定位按 IMOSS 先于子项目的顺序**。当 IMOSS 和某子项目都存在同一 ID 的任务（例如 IMOSS/013 和 onoai/013 同时存在），在子项目里 commit `feat(013)` 会错误地命中 IMOSS 的那个。

**修法思路**：
- `scripts/hooks/post-commit-audit.mjs` 的 `findTaskDir` 接受一个 `preferredProject` 参数
- shell hook 根据 `$REPO_ROOT` 判断当前处在哪个项目（IMOSS 根 → "IMOSS"，否则用 `basename $REPO_ROOT` 或遍历 registry 反查），作为 preferredProject 传入
- findTaskDir 先在 preferredProject 里找，找不到再 fall back 到其它搜索顺序

由于 onoai 清理后没有 ID 冲突，这个限制不会立即造成误告警，所以作为独立 follow-up 记录不在 3ebac51 里一起修。

## 对未来 agent 的规则

**面对子项目的任务状态清理时，单独看 `.imoss/tasks/active/**/context.json` 是不够的。必须同时跑 `git log --all --oneline | grep -E "feat\(NNN\)"`（或对应 scope 语法）来看代码是否真的落过。** 状态机和 git 历史脱节在 IMOSS 里不是边缘情况，是会重复发生的结构性问题——即使 3ebac51 修完子项目 hook 覆盖，也只是"未来新 commit 会被审计"，**历史已脱节的任务不会被追溯**，清理时仍要交叉核对。

这条规则应该叠加到现有的 "状态文件核对铁律"（feedback_handoff_verification）之上：两条都是"信任其他状态文件前，先信 git log"。

## 参考

- `/home/devuser/dev/imoss/projects/onoai/.imoss/tasks/done/` — 2026-04-09 cleanup 后归档的 11 个任务
- commit `6a28c91` (onoai) — cleanup commit
- commit `3ebac51` (IMOSS) — hook 覆盖修复
- `.imoss/knowledge/lessons/2026-04-task-misplacement-and-state-drift.md` — 本 lesson 的前置
- `scripts/install-git-hooks.mjs` + `scripts/git-hooks/post-commit` — 修复后的安装脚本与 hook
- `scripts/hooks/post-commit-audit.mjs` — audit 脚本（仍需后续修 ID 冲突 fallback）

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-04-11-telegram-state-dir-isolation|Telegram plugin 共享默认 state dir + last-restart-wins + wrapper bash env 丢失三重坑]]
