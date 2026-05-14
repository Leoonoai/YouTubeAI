---
status: active
createdAt: 2026-05-05T00:00:00.000Z
scope: codex-review-routing
confidence: 90
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Codex review window routing

## Problem

On 2026-05-05, the `imoss-codex` control window accepted a `xuexi` project plan-review prompt, entered the `xuexi` project, and began reading task 029 files. That was wrong: `imoss-codex` is the IMOSS/P001 Codex agent window, not a shared reviewer pool window.

## Rule

When a sub-project task needs Codex plan review:

- Do not perform the review inside `imoss-codex`.
- Use the registered reviewer pool windows: `imosscodex`, `codex2`, and `codex3`.
- If the request is to operate on the project as an agent rather than reviewer pool, use that project's Codex window, for example `xuexi-codex`.
- If `imoss-codex` accidentally enters a sub-project, immediately run `exit-project`, re-enter IMOSS/P001, and record the routing mistake before continuing.

## Recovery Applied

The mistaken `xuexi` binding was exited, and `imoss-codex` was restored to `P001 IMOSS`.
