---
id: lesson-2026-04-11-imoss-root-vs-project-task-root
title: >-
  Session registry root ≠ active project task root — cross-project current-task
  resolver
status: active
confidence: 95
tags:
  - hooks
  - task-resolver
  - cross-project
  - session-state
  - resolver-pattern
createdAt: 2026-04-11T00:00:00.000Z
sourceTasks:
  - '130'
  - '131'
relatedLessons:
  - 2026-04-11-telegram-state-dir-isolation
fixedIn:
  - commit: TBD
    date: 2026-04-11T00:00:00.000Z
    summary: >-
      131 新增 getCurrentTaskRef 共享 resolver，修 stop.cjs / task-workflow.cjs /
      user-prompt-submit.cjs 的 cross-project fallback
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 事件

130 号任务清理 129 跨窗口串号 handoff 污染时，在"baima 窗口的 `last-handoff-baima.json` 被重新污染"的尴尬复现中发现：**129 只修了 legacy fallback 一族的 bug，还有另一族同类问题藏在 hook 的 path resolution 层**。

具体重现：
1. 我在 imoss 窗口编辑 130 plan.md
2. 不相关的 baima claude（自己的 agent turn 结束）触发 stop hook
3. baima 的 stop hook 读 `active-baima.json.currentTaskId=019`
4. 去 **imoss 根** 的 `.imoss/tasks/active/` 找 `019-*` → 找不到
5. fallback 到 `findActiveTask(imoss 根)` → 返回最近编辑的任务 = imoss 的 **130**
6. 写 `last-handoff-baima.json.lastTaskId=130`

"imoss 最近编辑的任务"被写进了 baima 的 handoff。这与 129 的 legacy fallback bug 同族，但是不同层：129 修的是 `readActiveSession` 的读取路径，131 修的是 hook 代码对"任务目录在哪"的假设。

## 根因

**Session registry root 和 active project task root 是两个概念，hook 代码把它们混用了：**

- **Session registry root** = imoss 根的 `.imoss/sessions/` —— 所有窗口的 per-window session 文件都集中存在这里。`findImossDir(cwd)` 返回的 imoss 根 `.imoss`，这本身没错。
- **Active project task root** = 根据 `session.activeProjectPath` 动态决定的 `<activeProjectPath>/.imoss/tasks/active/` —— 子项目（baima/onoai/...）的任务目录**不在** imoss 根里，而在各自项目根里。

反模式：
```js
// 错：把 imossDir 当成任务根
const activeDir = path.join(imossDir, 'tasks', 'active');
const dir = fs.readdirSync(activeDir).find(d => d.startsWith(currentTaskId + '-'));
```

这行代码在 imoss 窗口里工作（因为 imoss 的任务确实在 imoss 根），但在 baima/onoai 窗口里必然失败——找不到就 fallback 到 imoss 根的 mtime 最新任务，把其他项目的工作污染到当前窗口。

反模式在 5 个 hotspot 同时存在：
1. `optional/hooks/claude/stop.cjs:107-123`（handoff writer）
2. `optional/hooks/lib/task-workflow.cjs:102-129`（workflow state resolver）
3. `optional/hooks/claude/user-prompt-submit.cjs:133-169`（consolidate cooldown）
4. `optional/hooks/claude/user-prompt-submit.cjs:218-261`（workflow hint injection）
5. `optional/hooks/claude/user-prompt-submit.cjs:293-315`（110 硬阻断判据）

第 5 个是最严重的：110 号规则的"无活跃任务时阻断源码写"判据，在子项目窗口里会把 baima/onoai 的合法任务误判为"不存在"，理论上让子项目根本无法开发。只是因为 129 之前 wrapper bash env 丢失让 stable-key 失效走 legacy fallback 串到 imoss 任务号，这个 bug 被掩盖了。

## 修复（131）

**统一的 current-task resolver**：`getCurrentTaskRef(imossDir)` 放在 `optional/hooks/lib/session-key.cjs`，契约是：

```js
{
  taskId: string,       // e.g. "019"
  taskDir: string|null, // absolute, null means bound but dir unresolved
  projectRoot: string,  // absolute path to active project root (NOT .imoss)
  session: object       // full active session object
}
```

解析规则：
1. 没 session / 没 currentTaskId → `null`（调用方可用自己的 lobby fallback）
2. 没 activeProjectPath（旧 global session）→ projectRoot = `path.dirname(imossDir)`（向后兼容）
3. 有 currentTaskDir → `path.resolve(projectRoot, currentTaskDir)`，**必须**校验 `startsWith(projectRoot + sep)`（防 `../../etc` 脏数据逃逸）且文件存在
4. 没 currentTaskDir → 在 `projectRoot/.imoss/tasks/active/` 按 taskId 前缀查
5. **关键约束**：有 currentTaskId 但任务目录找不到 → 返回 `{ taskId, taskDir: null, ... }`，调用方**绝不**可以 fallback 到 imoss 根 mtime 最近任务

调用方改造原则：
- `taskRef?.taskDir` 存在 → 用它
- `taskRef` 存在但 `taskDir: null` → 返回 null / false（让调用方知道"绑定 id 存在但没有可用的任务目录"，不要做任何项目外的 fallback）
- `taskRef` 为 null → 调用方可以用自己的 lobby fallback（mtime / findActiveTask）

## 教训要点

1. **Session state 和 task state 是两个独立的身份维度**。per-window session 回答"这个窗口当前绑定哪个项目 + 哪个任务"；task state 回答"这个任务在哪，它是什么状态"。把前者的结果直接拿去在后者的"默认根"下查找，就是 131 这类 bug。

2. **"`fallback` 到 mtime 最新" 是典型的隐藏炸弹**。它在单项目场景下完美工作，然后在多项目场景下**悄悄**把别人的数据串到你的 handoff。凡是 `findActiveTask(imossDir)` / `findNewestActivePlan(tasksBase)` 的 fallback 都必须严格条件限定——"只在没有任何 binding 时才允许"。有 binding 说明调用方真的在做某件事，找不到应当报错/返回 null，不应该默认去找别的任务。

3. **同一个 bug family 可能在多个 hotspot 同时存在**。131 的 5 个反模式写法几乎一模一样，都是"getCurrentTaskId + 在 imoss 根查找"。共享 resolver 的价值不只是减少重复，更是**统一错误处理契约**——让所有 hook 用同一套"绑定 id 存在但目录缺失"的语义，不可能某处漏改。

4. **path resolver 必须做输入消毒**。`currentTaskDir` 来自 per-window session 文件，原则上是受控写入，但脏数据（比如用户手工编辑、老版本写入、恶意注入）能让 `path.join` 逃出 `projectRoot`。一定要 `path.resolve` + `startsWith(root + sep)` 双重校验。

5. **工具语义分层要清楚**。`findActiveTask` / `findNewestActivePlan` 是通用 mtime 工具，语义是"给定目录找最近修改的任务"。调用方的错是让它们帮自己推理"哪个项目的任务"——这超出了工具的语义。131 的修法是**不改工具**，只改调用方让它们传入正确的目录（或不调用）。

6. **测试时必须区分 `--test` 模式和 stdio 模式**。Codex R1 审查时指出，`user-prompt-submit.cjs --test` 不会走完 `workflowHint` 的注入路径，只能用 `printf '{"prompt":"..."}' | IMOSS_SESSION=X node ...` 的 stdio 模式才能验证完整行为。这是 hook 测试的常见陷阱。

7. **Bug family 里"最严重的"往往被"次严重的"掩盖**。110 硬阻断判据的 bug 比 handoff 串号**严重得多**（前者让子项目窗口根本不能开发，后者只是 handoff 文件脏了），但用户很长时间只报告后者——因为上游的 wrapper bash env 丢失（129 bug）让严重的 bug 沿着 legacy fallback 错位掩盖了。修复顺序应该是 upstream → downstream，每修一层就暴露更底下的一层。

## 验证证据

**Unit tests (/tmp/imoss-131-test fixture)**：
- Test 1：baima session + 有效 currentTaskDir → 正确解析到 `projects/baima-juben/.imoss/tasks/active/019-baima-task`
- Test 2：坏 currentTaskDir → 回退到项目自身的 prefix scan，**不** fallback 到 imoss 根
- Test 3：有 currentTaskId 但项目内找不到 → 返回 `{ taskId, taskDir: null }`
- Test 4：无 session → null
- Test 5：`currentTaskDir: "../../../../etc"` → 校验失败，回退到 prefix scan（不逃逸出 projectRoot）

**Real-world tests**：
- Test 6/13：imoss IMOSS_SESSION 窗口的 stop hook → 写 `last-handoff-imoss.json` = 当前 imoss 任务（不受影响）
- Test 7/14：baima IMOSS_SESSION 窗口的 task-workflow 解析 → 返回 baima 项目的真实 019 任务（`projects/baima-juben/.imoss/tasks/active/019-深度分析-baima-juben-创作流程与-cli-提示词`），status=investigating 是真实状态
- Test 9：baima stdio user-prompt-submit → workflowHint 指向 baima 项目 019 而非 imoss 131
- Test 12：cooldown key `buildCooldownKey("IMOSS", "019") = "IMOSS-019"` ≠ `buildCooldownKey("baima-juben", "019") = "baima-juben-019"`，项目命名空间隔离

**Validation after fix**：
```bash
printf '{"cwd":"/home/devuser/dev/imoss","session_id":"131-validation-baima"}' | \
  IMOSS_SESSION=baima node optional/hooks/claude/stop.cjs
# → writes last-handoff-baima.json:
#   "lastTaskId": "019",
#   "lastTaskDir": "projects/baima-juben/.imoss/tasks/active/019-深度分析-..."
```

before 131：same command would write `lastTaskId=131` (imoss 的最近任务) → 跨项目污染。

## 后续

- 131 还需要实战验证：等 baima/onoai claude 自己跑 agent turn 时观察 handoff 文件是否继续保持正确
- cooldown 状态文件命名已改，旧的 `task-consolidate-last-<id>.json` 会变成孤儿文件（不造成问题，下次触发时会生成新的 project-scoped 文件）
- `findActiveTask` / `findNewestActivePlan` 的调用方已全部改完，但如果将来再新增 hook，必须用 `getCurrentTaskRef` 而不是老 API
