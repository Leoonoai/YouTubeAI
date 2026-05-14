---
title: IMOSS 故障排查指南
type: solution
status: active
confidence: 85
tags:
  - troubleshooting
  - faq
  - ops
createdAt: 2026-04-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# IMOSS 故障排查指南

## Dashboard 相关

### Dashboard 打不开 (localhost:3000)
1. 检查服务是否运行: `ps aux | grep tsx`
2. 检查端口占用: `lsof -i :3000`
3. 重启: `cd dashboard && npx tsx server/index.ts`
4. 如果端口被占: `fuser -k 3000/tcp && npx tsx server/index.ts`

### Dashboard 数据不更新
1. 点击页面上的「刷新」按钮
2. 点击「同步数据库」按钮
3. 或手动执行: `node scripts/sync-rebuild.mjs`

### Dashboard 样式丢失（所有元素堆叠）
原因: Tailwind v4 需要 @tailwindcss/vite 插件
修复: 检查 dashboard/vite.config.ts 是否包含该插件

## tmux-webview 相关

### tmux-webview 打不开 (localhost:8765)
1. 检查: `ps aux | grep server.py`
2. 重启: 
   ```bash
   cd tools/tmux-webview
   export TMUX_WEBVIEW_HOST=0.0.0.0
   export TMUX_WEBVIEW_PORT=8765
   export TMUX_WEBVIEW_USER=imoss
   export TMUX_WEBVIEW_PASS=<你的密码>
   nohup python3 server.py > /dev/null 2>&1 &
   ```

## Hook 相关

### Hook 不触发
1. 检查 settings.json: `cat ~/.claude/settings.json | grep hooks`
2. 确认 hook 脚本有执行权限: `chmod +x optional/hooks/claude/*.cjs`
3. 测试单个 hook: `node optional/hooks/claude/user-prompt-submit.cjs --test "帮我review"`

### 进入项目失败
1. 确认 active-session.json 状态: `cat .imoss/active-session.json`
2. 清除旧会话: `rm .imoss/active-session.json`
3. 重新进入: `node scripts/enter-project.mjs <项目名>`

## 知识库相关

### 知识提取失败
1. 检查 Claude Code session logs: `ls ~/.claude/projects/`
2. 手动执行: `node scripts/extract-knowledge.mjs --dry-run`
3. 检查 OpenAI API Key（知识精炼需要）

### 搜索结果不准确
1. 重建 FTS 索引: `node scripts/sync-rebuild.mjs`
2. 检查中文分词: jieba 是否安装 (`@node-rs/jieba`)
3. 尝试不同关键词（分词可能与预期不同）

## Git 相关

### sync-rebuild 后任务消失
原因: sync-rebuild 是全量 DROP+CREATE，会清除 DB 数据
解决: 任务数据来自 .imoss/tasks/ 文件，只要文件在就能重建

### 新任务不显示在 Dashboard
1. 确认 plan.md 有 YAML frontmatter（id, title, status）
2. 点击 Dashboard「同步数据库」按钮
3. 或执行: `node scripts/sync-rebuild.mjs`

## 性能相关

### 内存不足 (OOM)
1. 检查: `free -h`
2. 添加 swap: `sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile`
3. 减少并发 Claude Code 会话数

### Dashboard 加载慢
1. 检查 DB 大小: `ls -la dashboard/data/imoss.db`
2. 重建索引: `node scripts/sync-rebuild.mjs`
3. 超过 1000 条知识时考虑分页查询
