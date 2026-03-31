# Claude Code 2.1.88 综合分析概览

## 已完成
- 提取 npm 原包与 cli.js.map
- 从 sourcesContent 还原源码
- 剥离 node_modules，生成仅自有源码目录
- 基于目录与关键词完成模块梳理与敏感点扫描

## 推荐优先阅读
- src/commands/login/login.tsx
- src/commands/logout/logout.tsx
- src/services/oauth/index.ts
- src/utils/auth.ts
- src/utils/permissions/PermissionMode.ts
- src/utils/permissions/permissions.ts
- src/utils/permissions/yoloClassifier.ts
- src/tools/BashTool/BashTool.tsx
- src/tools/BashTool/bashSecurity.ts
- src/tools/AgentTool/AgentTool.tsx
- src/tools/AgentTool/forkSubagent.ts
- src/services/mcp/MCPConnectionManager.tsx
- src/commands/mcp/mcp.tsx
- src/bridge/bridgeMain.ts
- src/remote/RemoteSessionManager.ts
- src/services/analytics/index.ts
- src/services/analytics/datadog.ts
- src/utils/telemetry/events.ts
- src/components/AutoUpdater.tsx
- src/utils/nativeInstaller/installer.ts
- src/utils/model/providers.ts
- src/utils/model/antModels.ts
- src/commands/bughunter/index.js
- src/commands/mock-limits/index.js

## 初步判断
- 这是一个功能非常完整的 Claude Code 客户端，不只是简单终端封装。
- 权限体系很重：同时存在 UI 审批、规则引擎、路径校验、危险命令分类和 bypass/yolo 相关逻辑。
- agent / teammate / swarm 是核心设计，不是实验性一角。
- MCP、remote bridge、direct connect、session ingress 说明它具备跨进程/跨会话/远端接入能力。
- 存在明显的 telemetry/analytics/experiment 基础设施。
- 存在较多 debug/内部命令痕迹，值得单独翻。
