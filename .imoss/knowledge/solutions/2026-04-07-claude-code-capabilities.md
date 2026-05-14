---
title: Claude Code 实战能力全景
type: solution
status: active
confidence: 90
source: task-070-research
tags:
  - claude-code
  - ecosystem
  - capabilities
  - mcp
  - plugins
createdAt: 2026-04-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Claude Code 实战能力全景

## 一、内置工具

| 工具 | 功能 |
|------|------|
| Bash | 执行任意 shell 命令 |
| Read | 读文件（含图片、PDF、Jupyter） |
| Write/Edit | 写文件/精确替换 |
| Glob/Grep | 文件名搜索/内容搜索(ripgrep) |
| WebSearch/WebFetch | 网页搜索/获取网页内容 |
| Agent | 派生子代理，独立上下文并行工作 |
| TodoRead/TodoWrite | 结构化任务追踪 |
| NotebookEdit | Jupyter notebook 编辑 |
| CronCreate/List/Delete | 会话级调度（/loop） |
| RemoteTrigger | 云端调度任务管理 |

## 二、四大扩展支柱

### MCP（Model Context Protocol）
- 连接外部系统的通用适配器
- 生态：770+ MCP servers（claudemarketplaces.com）
- 关键 MCP：GitHub(28K⭐)、Firecrawl(web抓取)、n8n(工作流)、Slack、Telegram

### Skills（技能）
- `.claude/skills/` 目录的 Markdown 文件 → `/slash-command`
- 生态：2,300+ skills
- 支持自动触发（按 description 匹配）或手动触发

### Hooks（钩子）
- 7 个事件点：PreToolUse/PostToolUse/UserPromptSubmit/SessionStart/Stop/PreCompact/PostCompact
- 4 种处理器：Command(shell)、Prompt(LLM)、HTTP(webhook)、Async(后台)
- 保证执行，不依赖模型行为

### Plugins（插件）
- 官方市场 120+ 插件
- 涵盖：code-review, security, frontend-design, LSP语言服务器(15种语言), Vercel, Supabase, Stripe 等

## 三、调度能力（三层）

| 层级 | /loop (会话级) | Desktop (桌面级) | Cloud (云端) |
|------|---------------|-----------------|-------------|
| 运行位置 | 当前会话 | 本机 | Anthropic 云 |
| 需要电脑开着 | 是 | 是 | 否 |
| 跨重启持久 | 否(除非durable) | 是 | 是 |
| 最小间隔 | 1 分钟 | 1 分钟 | 1 小时 |
| 本地文件访问 | 是 | 是 | 否(fresh clone) |

## 四、多 Agent 协作

- `claude -w task-name` 自动建 git worktree 隔离
- Anthropic 内部同时跑 10-15 个并行 session
- 社区工具：claude-squad(6.9K⭐) 管理多 agent
- claude-flow(29K⭐) 多 agent 编排 + 100+ MCP

## 五、真实用户案例

1. **产品经理做了 13 个项目**：iOS App 上架 App Store、足球预测 ML 模型、家谱追溯 17 世纪
2. **一人 DevOps**：零手动部署 AWS（S3+CloudFront），3 子 Agent 做安全审计/IaC/成本优化
3. **Reddit 实战**：3000 行 C# 重构、Lead Gen 扫描 GitHub、Playwright 自动文档
4. **个人自动化**：语音笔记 → Telegram → 转录 → 路由项目 → 自主执行 → Telegram 回报
5. **非编码**：数据分析 CSV、Remotion 视频、截图转代码、监控植物

## 六、SEO/营销/社媒 生态

| 项目 | 说明 |
|------|------|
| claude-seo | 19 子技能、12 subagent，覆盖技术 SEO/E-E-A-T/外链/本地 SEO |
| ai-marketing-claude | 15 技能：网站审计/文案/邮件/广告/竞品情报 |
| marketing-skills | 160+ 开源营销技能 |
| seomachine | SEO 博客内容工厂 |
| postiz (官方插件) | 社交媒体自动化，28+ 平台 |

## 七、能力边界

### 做得好
- 大规模代码重构（跨 10+ 文件）
- 从零搭建项目（Web/iOS/CLI/数据管道）
- DevOps 自动化（Terraform/Docker/AWS）
- 多 Agent 并行开发
- MCP 集成外部系统
- Hooks 安全防护和质量保障
- 非编码任务（数据分析/视频/文档/研究）

### 有限制
- 超长 session 上下文会耗尽（需 /compact）
- 不能看浏览器（需 Playwright 截图）
- 不支持交互式命令（git rebase -i 等）
- 费用控制困难（重度 session $5-15/h）
- 质量波动（同任务不同 session 表现差异大）

### 已知坑
- 2026-02 prompt 缓存 Bug（token 膨胀 10-20x，需新 session）
- RemoteTrigger API 有 500 错误（GitHub #43438）
- METR 研究：有经验开发者用 Claude Code 复杂任务反而慢 19%

## 八、关键 GitHub 仓库

| 仓库 | Stars | 用途 |
|------|-------|------|
| anthropics/claude-code | 101K | 本体 |
| affaan-m/everything-claude-code | 128K | 最全资源集 |
| obra/superpowers | 55K+ | 结构化开发方法论 |
| ruvnet/claude-flow | 29K | 多 agent 编排 |
| github/github-mcp-server | 28K | GitHub MCP |
| anthropics/claude-code-action | 6.9K | GitHub Actions |
| smtg-ai/claude-squad | 6.9K | 多 agent TUI 管理 |
| zilliztech/claude-context | 5.2K | 语义代码搜索 MCP |
| rohitg00/awesome-claude-code-toolkit | - | 135 agents+35 skills+150 plugins |

## 九、第三方目录

| 平台 | 规模 |
|------|------|
| claudemarketplaces.com | 2,300+ skills, 770+ MCP |
| anthropics/claude-plugins-official | 120+ 插件 |
| anthropics/skills | 官方技能库 |
| pulsemcp.com | MCP Server 目录 |
