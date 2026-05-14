---
title: Dashboard sendKey 路径不能加 verify+retry（Claude AI 建议文字会被误清）
type: lesson
status: active
confidence: 85
audience: work
tags:
  - dashboard
  - tmux
  - claude-tui
  - terminal-viewer
  - regression
source: task307
sourceTasks:
  - '307'
createdAt: 2026-05-10T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Dashboard sendKey 路径不能加 verify+retry

## 现象

307 给 dashboard `tmux.sendKey` 路径加了 "Enter + Claude/Codex executor + 有 pending text → verifyPromptSubmit 重试 (Enter, C-j, Enter)" 的逻辑，复用 302 给 sendText 的 verify 库。改完看似 fix 了"按 ⏎ 提不了悬空文字"的问题，实际引入更严重的 regression：

**Claude Code TUI 任务结束后会在 ❯ 行显示"AI 建议的下一步追问"**（类似 fish/zsh autosuggestion 的 ghost text）。这段建议文字落在 `^❯ ...` 行 → sendKey 的本地 helper `extractPromptLineUserInput` 把它识别为 pending text → 走 verify+retry 路径 → 多发的 `C-j` (newline) / 第二次 `Enter` 把刚刚提交的内容打散或清掉。

回退后单发 Enter（192 时代行为）反而正确——Claude TUI 自己会把 ❯ 上的建议直接当用户输入提交。

## 根因

1. **`extractPromptLineUserInput` 不能区分"用户真的敲进去的文字" vs "Claude 自动展示的 ghost suggestion"**。两者在 `tmux capture-pane` 输出里完全同形（都是 `^❯ <text>` 行）。pane 文本层面没办法判别，除非引入完全不同的状态机。

2. **verifyPromptSubmit 的契约假设是"caller 已发了首次 Enter，需要观察 fingerprint 是否被清掉"**。但 Claude TUI 提交后会把用户输入回显成消息块（带 `> ` 引用），fingerprint 还在 pane 里 → verify 误判 still-pending → 多发 retry key 把后续操作打乱。

3. **sendText 路径 OK 是因为它有完整的 pre-clear**：先 `clearPendingInputBeforeSend` 清掉输入框残留，再 paste 自己写入的文字，fingerprint 是自己生成的 → 提交后 fingerprint 唯一对应自己写的内容。sendKey 路径没有 pre-clear / 不知道"用户预期提交什么"，把任意 ❯ 行文字当 fingerprint 是不安全的。

## 教训

**dashboard `tmux.sendKey` 必须保持 dumb passthrough**：直接 `tmux send-keys -- <key>`，不做"智能"。

- 想让 ⏎ 在 web 端可靠提交 TUI 里悬空文字 → 应该走 sendText 路径（前端在按钮 handler 里检测 ❯ 行，先把文字读出来再 sendText 提交），而不是改 sendKey 服务端
- 任何"server 端读 pane 文本判断该不该重发 key"的方案都要先解决"用户输入 vs ghost suggestion 不可区分"这个根问题，不是直接套 verify 库

## 何时复用此教训

- 新增"自动 verify + retry"逻辑时：先问 fingerprint 是不是能 100% 唯一锚定到"用户预期被提交的文字"。锚不准就别 verify
- 改 dashboard 里任何"server 帮 user 智能判断按了什么键意味着什么"的代码：默认拒绝，让前端把语义说清楚再传过去
- 评估"复用现有 verify 库"时，先看现有 caller（sendText）满足哪些前提（pre-clear / 自己写的 fingerprint），新 caller 是否同样满足

## 相关

- 任务 307（被本 lesson revert）
- 决策 302（sendText 加 verify 是合理的，因为有 pre-clear + 自己写 fingerprint）
- `dashboard/server/lib/tmux-submit-verify.mjs:188` verifyPromptSubmit 契约
