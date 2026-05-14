---
id: decision-2026-05-research-pool-first-architecture
title: 研究系统改"池先 + 执行端决定 task 归属"，imoss-cc 只 triage 不 task-create
status: active
confidence: 90
tags:
  - research
  - architecture
  - pool
  - worker
  - triage
source: task-343
createdAt: 2026-05-13T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

把研究系统的入口从 "task 先建" 反转为 "池先 + 执行端决定归属"：

- **imoss-cc（claude-code / codex 主对话窗口）只做 triage**：把用户讨论的信息整理成 `triagedTopic` + `materialBody`，调 `research-pool-enqueue.mjs` 入池，**不再 task-create，不再 dedup**
- **marker schema v3** 允许 `taskId: null`，加 `materialBody` + `triagedTopic` 字段
- **Mac worker 自治**：claim 后 if taskId==null → 跑 `research-query-dedup.mjs` → strong band 复用现有 R-NNN / weak+0 命中新建 → `materializeTaskFromPoolEntry` patch 新 task plan.md（写 originPoolEntry + ## 信息池输入材料 段）→ `writebackTaskId` 回写 → 才 dispatch iterate
- **Telegram 通知头区分** 🆕 新建 / 🔄 关联 / 🔬 task-first（老 flow）

仅在用户**明确**说"入研究池 / 加入课题池 / 开研究课题"等池入口词时才走此 flow。默认 "研究一下 X / 给 X 做调研" 仍在 imoss-cc 主窗口直接 WebFetch 调研，不入池。

显式 "iterate R-NNN" / 234 cron 注入 "迭代 NNN" 时 imoss-cc 仍直接带 taskId enqueue，跳过 worker triage 走老 task-first flow。

## 背景

342 之前的设计是 imoss-cc 在对话时**捆绑** task-create + dedup + enqueue 三件事。用户 5/13 反馈："信息池本身**没有用户价值**——它只是跨机异步调度队列的实现细节。"

用户提出新设计："claude code 或 codex 窗口讨论信息归类整理好进研究信息池，然后研究窗口读取信息池决定添加到现有课题任务还是新建课题任务"。

核心洞察：
- triage（决定主题）和 execution（执行迭代）是不同职责，应该分离
- imoss-cc 是 idea triage 层（轻量：听用户讲 → 整理成池条目 → 入池）
- 研究窗口有完整上下文（knowledge / 现有 task 列表）能更好地决定 task 归属
- 信息池才有"first-class 意义"：已 triage 的待研究条目仓库，不是隐性队列

## 理由

1. **职责分离**：triage 与 execution 解耦，多 agent 都能往池里加（claude-code + codex），执行收敛在 Mac
2. **信息池有真意义**：从"隐性调度队列"升级为"已分类待研究内容仓库"，processed/ 历史给出研究决策史
3. **Worker 自治**：研究窗口拿全部上下文做匹配判断，比 imoss-cc 在对话流水线里硬塞 dedup 更可靠
4. **同一 task 多 RP 自然形成**：每次 enqueue 独立 seqId（342 已落地），同主题第二次入池 dedup 命中已建 task → 自然 add-to-existing
5. **保留 backward compat**：marker payload taskId 非空时跳 worker dedup 直接 dispatch；imoss-cc "iterate R-NNN" 显式入口不变

## 操作规范

**imoss-cc 收到"入池/课题"明确触发词时**：

```bash
node scripts/research-pool-enqueue.mjs --mode iterate \
  --triaged-topic "<主题摘要 ≤50 字>" \
  --material "<用户讨论原文 / 整理后内容>" \
  --requested-by "${IMOSS_SESSION:-imoss-cc}"
```

不传 `--task`，marker payload taskId=null。

**imoss-cc 收到"iterate R-NNN" 显式入口时**（老 flow）：

```bash
node scripts/research-pool-enqueue.mjs --task NNN --topic "..." --mode iterate --direction "..."
```

**Worker 端**（自动）：
- `claimedPayload.taskId == null` → triage flow（dedup → 决策 → task-create or 复用 → materialize → writebackTaskId → dispatch）
- `claimedPayload.taskId != null` → 直接 dispatch（跳过 triage，等同 342 行为）

**默认非池入口**：用户说"研究一下 X / X 怎么用" 等模糊触发词 → imoss-cc 主窗口直接 WebFetch + 写报告，**不入池**。

## 参考

- task 343 plan.md / progress.md：完整 6 阶段实施记录
- task 342：342 marker schema v2（seqId + processed/）— 343 在它之上加 v3 字段
- task 318：`research-query-dedup.mjs` + `inbox-classify.cjs` 提供 worker 端 dedup 算法（直接 require memory-corpus 不依赖 MCP）
- memory `feedback_research_marker_queue.md`：5/11 用户 + 5/13 343 双层语义边界（任务级 vs 池入口）
- 新 helper `scripts/lib/research-pool-task-materialize.cjs`：worker 新建 task 后 patch plan.md 写 originPoolEntry + materialBody section
- `.imoss/skills/imoss-query-augment.json` 顶部新增"343 池先架构"段
