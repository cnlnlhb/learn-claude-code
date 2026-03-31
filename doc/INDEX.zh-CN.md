# 文档索引

[返回 README](../README.zh-CN.md) · [English Index](./INDEX.en.md)

这个页面是本仓库研究文档的结构化导航中心。

---

## 从这里开始

如果你第一次看这个仓库，建议按这个顺序：

1. [README](../README.zh-CN.md)
2. [认证登录](./authentication-login.zh-CN.md)
3. [权限风控](./permissions-risk-control.zh-CN.md)
4. [多 Agent](./multi-agent.zh-CN.md)
5. [MCP](./mcp.zh-CN.md)
6. [远程 / 桥接](./remote-bridge.zh-CN.md)
7. [埋点 / Telemetry](./telemetry.zh-CN.md)
8. [更新 / 安装](./update-install.zh-CN.md)

---

## 推荐阅读路径

### 路径 A —— 把 Claude Code 当作一个产品运行时来理解

1. [认证登录](./authentication-login.zh-CN.md)
2. [权限风控](./permissions-risk-control.zh-CN.md)
3. [远程 / 桥接](./remote-bridge.zh-CN.md)
4. [埋点 / Telemetry](./telemetry.zh-CN.md)
5. [更新 / 安装](./update-install.zh-CN.md)

原因：
- 认证解释系统如何识别“你是谁”
- 权限解释系统允许你做什么
- 远程 / 桥接解释执行如何离开本机
- telemetry 解释行为如何被观察、控制和 rollout
- 更新 / 安装解释客户端如何被交付与迁移

### 路径 B —— 把 Claude Code 当作一个编排系统来理解

1. [多 Agent](./multi-agent.zh-CN.md)
2. [MCP](./mcp.zh-CN.md)
3. [远程 / 桥接](./remote-bridge.zh-CN.md)
4. [权限风控](./permissions-risk-control.zh-CN.md)

原因：
- 多 Agent 解释 worker 编排
- MCP 解释扩展与能力注入
- 远程 / 桥接解释 session transport 与控制通道
- 权限解释这些能力是如何被约束的

### 路径 C —— 把 Claude Code 当作一个安全执行客户端来理解

1. [认证登录](./authentication-login.zh-CN.md)
2. [权限风控](./permissions-risk-control.zh-CN.md)
3. [远程 / 桥接](./remote-bridge.zh-CN.md)
4. [埋点 / Telemetry](./telemetry.zh-CN.md)

原因：
- 认证控制身份
- 权限控制本地执行
- 远程 / 桥接控制远端执行
- telemetry 能看出 enforcement、rollout 与 incident response 的模式

---

## 模块入口

### 认证登录 / Authentication & Login
- [中文](./authentication-login.zh-CN.md)
- [English](./authentication-login.en.md)

主题：
- OAuth + PKCE
- Claude.ai / Console 双入口
- managed auth context
- token 来源层级
- trusted device 联动

### 权限风控 / Permissions & Risk Control
- [中文](./permissions-risk-control.zh-CN.md)
- [English](./permissions-risk-control.en.md)

主题：
- permission modes
- allow/deny/ask 规则系统
- classifier 驱动审批
- managed policy override
- Bash / zsh 风险分析

### 多 Agent / Multi-Agent
- [中文](./multi-agent.zh-CN.md)
- [English](./multi-agent.en.md)

主题：
- AgentTool
- fork subagent
- teammate / swarm runtime
- task list
- mailbox 与 leader approval bridge

### MCP
- [中文](./mcp.zh-CN.md)
- [English](./mcp.en.md)

主题：
- MCP 连接生命周期
- auth 与 transport
- official registry
- tools / commands / resources
- 受策略控制的扩展模型

### 远程 / 桥接 / Remote / Bridge
- [中文](./remote-bridge.zh-CN.md)
- [English](./remote-bridge.en.md)

主题：
- bridge worker
- session ingress auth
- trusted device
- direct connect
- 远程审批 / 控制回流

### 埋点 / Telemetry
- [中文](./telemetry.zh-CN.md)
- [English](./telemetry.en.md)

主题：
- analytics 入口层
- Datadog 与 1P event logging
- GrowthBook
- metadata enrich
- 隐私控制与 kill switch

### 更新 / 安装 / Update / Install
- [中文](./update-install.zh-CN.md)
- [English](./update-install.en.md)

主题：
- auto updater
- native installer
- npm global/local 迁移
- package manager 检测
- rollout/version gate

---

## 仓库中的辅助材料

- [README（英文）](../README.md)
- [README（中文）](../README.zh-CN.md)
- [npm-original/](../npm-original/)
- [source/](../source/)

---

## 后续值得补的专题

这个仓库后面很适合继续补：

- hidden commands / feature flags
- 权限绕过链路
- remote session 协议细节
- telemetry 隐私边界
- MCP auth 深挖
