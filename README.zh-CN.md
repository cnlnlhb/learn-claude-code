# learn-claude-code

这个仓库保存了对 `@anthropic-ai/claude-code@2.1.88` 的本地研究副本，主要包括：

1. **npm-original/**
   - 从 npm 下载的原始包与关键产物
   - 包含用于还原源码的 `cli.js.map`

2. **source/**
   - 从 `cli.js.map` 中的 `sources` + `sourcesContent` 还原出的源码树
   - 当前仓库保留的是 Claude Code 自有源码为主的整理版本（已去除大部分第三方 `node_modules`）

3. **doc/**
   - 各类分析文档
   - 包括模块梳理、结构分析、敏感点扫描，以及按专题拆分的中英文报告

## 目录结构

- `npm-original/` — npm 原始文件与关键包内容
- `source/` — 解析还原后的源码
- `doc/` — 分析文档
- `README.md` — 英文说明
- `README.zh-CN.md` — 中文说明

## 模块分析导航

- 认证登录 / Authentication & Login
  - 中文：`doc/authentication-login.zh-CN.md`
  - English: `doc/authentication-login.en.md`

- 权限风控 / Permissions & Risk Control
  - 中文：`doc/permissions-risk-control.zh-CN.md`
  - English: `doc/permissions-risk-control.en.md`

- 多 Agent / Multi-Agent
  - 中文：`doc/multi-agent.zh-CN.md`
  - English: `doc/multi-agent.en.md`

- MCP
  - 中文：`doc/mcp.zh-CN.md`
  - English: `doc/mcp.en.md`

> 后续还会继续补充：远程/桥接、埋点、更新/安装等模块的中英文专题分析。

## 说明

- 包名：`@anthropic-ai/claude-code`
- 版本：`2.1.88`
- 源码还原依据：`cli.js.map` 中内嵌的 `sources` 与 `sourcesContent`
- 这个仓库的目标是便于学习、审阅和结构化分析，不是官方源码镜像

## 当前观察

从源码结构看，Claude Code 2.1.88 并不是一个简单的终端包装器，而是包含了：

- 完整的 OAuth / API Key 认证体系
- 权限审批与风险控制机制
- 多 Agent / teammate / swarm 协作能力
- MCP 集成
- 远程控制 / bridge / direct connect 能力
- telemetry / analytics / experiments 基础设施
- 自动更新与本地安装逻辑
