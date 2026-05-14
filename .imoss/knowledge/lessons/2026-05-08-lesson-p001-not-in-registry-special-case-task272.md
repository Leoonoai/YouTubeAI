---
title: P001 不在 registry，凡是 projectId-resolution 都得 special-case
type: lesson
status: active
confidence: 85
tags:
  - registry
  - project-resolution
  - p001
  - special-case
source: task-272
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# P001 不在 registry，凡是 projectId-resolution 都得 special-case

## 教训

新写涉及 `projectId → 项目根目录` 解析的 helper 时，**必须**给 P001 加 special-case：
P001 永远不在 `.imoss/registry.json`，纯按 `IMOSS_ROOT` 解析。任何只查 registry 的
解析逻辑会对 P001 直接失败。

同时 P001 root 也是"主仓 + 子仓库容器"，扫描 P001 必须额外跳过：
- 顶层 `projects/`（每个子仓库各自有自己的代码图）
- `.imoss/{tasks, knowledge, references, memory, inbox, sessions, logs, tmp, gateway}`
  （知识语料 / 运行时状态，混进代码图就是噪音）
- 保留 `.imoss/rules`（spec 也是项目代码）

## 原因

273 调研发现 `optional/mcp/server.mjs:getProjectPath(projectId)` 只查 registry —
P001 调用它会 throw，调用方一直手工写 `if projectId === 'P001' use IMOSS_ROOT else
查 registry` 这种条件。把这个判断**封装在 helper 里**而不是让每个调用方各写一遍。

P001 special-case 的另一面：扫文件系统时如果不跳 `projects/`，会把所有子仓库的
JSON / TS 文件都聚合进主仓 map — 总数从 ~420 跳到 8000+，且 ownership 跨仓库无意义
（git blame 在主仓跑不到子仓库的 commit）。Codex R1 第一时间拍出这个隐患。

## 后果（如果忽略）

- helper 对 P001 直接 throw → 调用方都要包 try/catch + fallback，分散逻辑
- project-map 不跳 projects/* → 输出量爆炸 + 数据无意义（混合多个 git history）
- project-map 不跳 .imoss/{tasks, knowledge, ...} → 把 200+ knowledge md 当代码扫，
  把 1000+ 任务 plan 当代码扫，污染代码图 retrieval

## 正确做法

```js
// scripts/lib/code-graph.cjs:resolveProjectRoot
function resolveProjectRoot(projectId, opts = {}) {
  const imossRoot = opts.imossRoot || IMOSS_ROOT_DEFAULT;
  // P001 special-case 直接写在 helper 里
  if (projectId == null || projectId === '' || projectId === 'P001') {
    return { projectId: 'P001', root: imossRoot };
  }
  // P00X 走 registry + 路径越界校验
  const project = registry.projects.find(p => p.id === projectId);
  if (!project) throw new CodeGraphError('PROJECT_NOT_FOUND', ...);
  // ...
}

// 扫描时按 projectId 决定 ignore 规则
function shouldSkipDir(absPath, root, projectId) {
  if (COMMON_IGNORE_DIRS.has(basename(absPath))) return 'common-ignore';
  if (projectId === 'P001') {
    if (rel === 'projects') return 'p001-projects';
    if (rel.startsWith('.imoss/')) {
      const subdir = rel.slice('.imoss/'.length).split('/')[0];
      if (IMOSS_KNOWLEDGE_RUNTIME_DIRS.has(subdir)) return 'p001-imoss-runtime';
    }
  }
  return null;
}
```

通用规则：**当一个常量（IMOSS_ROOT / P001 / 主仓）在系统里有特殊地位**，把
special-case 封装在最底层 helper，不要让每个调用方各写一遍 if-else。

## 参考

- 实现: `scripts/lib/code-graph.cjs:resolveProjectRoot` + `shouldSkipDir`
- 测试覆盖: `scripts/code-graph.test.mjs` "P001 special-case" describe 块
- 相关决策: [[2026-05-08-decision-code-graph-realtime-not-sqlite-task272]]
