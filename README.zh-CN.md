# learn-claude-code

[English README](./README.md)

一个围绕 `@anthropic-ai/claude-code@2.1.88` npm 发布包做的研究型仓库。

这个仓库基于**已公开发布的 npm 包**、其自带 sourcemap，以及从 sourcemap 中恢复出的源码文件构建而成。它的目标是用于逆向分析、架构研究和结构化文档整理，**不是官方源码镜像**。

---

## 这个仓库包含什么

这个仓库大致分成四层：

1. **原始 npm 产物**
   - 从 npm 下载到的原始包
   - 解包后的关键文件，例如 `cli.js`、`cli.js.map`、`package.json`

2. **解析恢复后的源码**
   - 基于 `cli.js.map` 恢复出的源码树
   - 重点保留 Claude Code 自有源码，尽量去掉第三方 `node_modules` 噪音

3. **研究分析文档**
   - 中英双语模块分析文档
   - 结构梳理、专题结论、阅读导航

4. **仓库级导航**
   - 英文 README
   - 中文 README
   - 推荐阅读顺序与主题索引

---

## 为什么会有这个仓库

因为 `@anthropic-ai/claude-code@2.1.88` 这个 npm 版本随包发布了一个可用的 sourcemap：

- `cli.js.map`

而且这个 sourcemap 内嵌了 `sourcesContent`，因此可以：

- 恢复出大量源码文件
- 直接观察发布构建中的架构设计
- 系统研究 Claude Code 在认证、权限、多 Agent、MCP、远程会话、埋点、安装更新等方面的实现思路

这个仓库就是把这些工作整理成一个更适合阅读的研究仓库，而不是让人直接去翻生硬的 npm tarball。

---

## 核心发现

`@anthropic-ai/claude-code@2.1.88` **并不是一个薄薄的终端壳**。

从恢复出的源码来看，它至少包含这些较完整的子系统：

- OAuth / API Key 认证体系
- 权限审批与风险控制体系
- 多 Agent / teammate / swarm 协作运行时
- 深度 MCP 集成
- 远程会话 / bridge / direct connect 执行能力
- telemetry / analytics / experiments 基础设施
- 安装、升级、迁移与分发逻辑

---

## 一眼看懂的关键结论

如果你只想先抓重点，先看这一段。

1. **身份状态是运行时控制面的核心输入**
   - 登录/登出影响的不只是 token
   - 还会联动 policy、feature gate、trusted device 和依赖服务状态

2. **权限系统不是二元开关，而是分层治理**
   - permission mode、allow/deny/ask、classifier、managed policy、shell 安全检查会一起起作用

3. **多 Agent 是一级能力，不是边缘实验**
   - 代码里有 teammate / swarm / task / mailbox / leader control 一整套协作机制

4. **MCP 已经高度产品化**
   - 连接管理、认证、registry、commands、resources、policy filtering、UI 都说明它是核心扩展总线

5. **远程执行是 session-centric 且高信任等级的能力**
   - bridge worker、session ingress auth、trusted device token、approval return path 都不是轻量设计

6. **telemetry 是运行时基础设施，不只是打日志**
   - Datadog、1P event logging、GrowthBook、OTel 风格路径各自承担不同角色，并带明显隐私边界

7. **交付系统明显处于迁移期**
   - 代码强烈表明它正在从 npm 分发向 native installer 分发演进

---

## 架构地图

一个便于快速建立直觉的简化心智模型：

```text
                         +----------------------+
                         |   Auth / Identity    |
                         | OAuth, API keys,     |
                         | managed context,     |
                         | trusted device       |
                         +----------+-----------+
                                    |
                                    v
+------------------+      +---------+----------+      +------------------+
| Update / Install | ---> |   Core Runtime     | <--- |   Telemetry      |
| npm, local,      |      | commands, tools,   |      | Datadog, 1P,     |
| native installer |      | state, sessions    |      | GrowthBook, OTel |
+------------------+      +----+-----------+---+      +------------------+
                                |           |
                                |           |
                                v           v
                     +----------+--+   +---+---------------+
                     | Permissions |   | Extensibility     |
                     | modes,      |   | MCP, plugins,     |
                     | rules,      |   | dynamic tools /   |
                     | classifiers |   | commands/resources|
                     +------+------+
                            |
                            v
                  +---------+-------------------+
                  | Orchestration / Execution   |
                  | multi-agent, swarm, remote, |
                  | bridge, direct connect      |
                  +-----------------------------+
```

这是一个简化图，但和当前恢复出来的包结构基本吻合。

---

## 按读者类型从哪里开始

### 如果你更关心安全
建议先看：
- [权限风控](./doc/permissions-risk-control.zh-CN.md)
- [认证登录](./doc/authentication-login.zh-CN.md)
- [远程 / 桥接](./doc/remote-bridge.zh-CN.md)
- [埋点 / Telemetry](./doc/telemetry.zh-CN.md)

### 如果你更关心产品架构
建议先看：
- [认证登录](./doc/authentication-login.zh-CN.md)
- [多 Agent](./doc/multi-agent.zh-CN.md)
- [MCP](./doc/mcp.zh-CN.md)
- [远程 / 桥接](./doc/remote-bridge.zh-CN.md)

### 如果你更关心逆向方法 / 实现策略
建议先看：
- [更新 / 安装](./doc/update-install.zh-CN.md)
- [MCP](./doc/mcp.zh-CN.md)
- [多 Agent](./doc/multi-agent.zh-CN.md)
- [埋点 / Telemetry](./doc/telemetry.zh-CN.md)

### 如果你更关心企业/托管运行时能力
建议先看：
- [认证登录](./doc/authentication-login.zh-CN.md)
- [权限风控](./doc/permissions-risk-control.zh-CN.md)
- [MCP](./doc/mcp.zh-CN.md)
- [远程 / 桥接](./doc/remote-bridge.zh-CN.md)
- [埋点 / Telemetry](./doc/telemetry.zh-CN.md)

---

## 模块卡片

### [认证登录](./doc/authentication-login.zh-CN.md)
[English](./doc/authentication-login.en.md)

身份体系、OAuth、managed auth context、token 来源与 trusted device 联动。

### [权限风控](./doc/permissions-risk-control.zh-CN.md)
[English](./doc/permissions-risk-control.en.md)

permission mode、规则系统、classifier、managed policy 与 shell 级安全分析。

### [多 Agent](./doc/multi-agent.zh-CN.md)
[English](./doc/multi-agent.en.md)

AgentTool、subagent、teammate/swarm runtime、task graph 与 leader control。

### [MCP](./doc/mcp.zh-CN.md)
[English](./doc/mcp.en.md)

连接生命周期、auth、registry、动态 tools / commands / resources。

### [远程 / 桥接](./doc/remote-bridge.zh-CN.md)
[English](./doc/remote-bridge.en.md)

bridge worker、session ingress auth、trusted device、direct connect、审批回流路径。

### [埋点 / Telemetry](./doc/telemetry.zh-CN.md)
[English](./doc/telemetry.en.md)

Datadog、1P logging、GrowthBook、metadata enrich 与隐私控制。

### [更新 / 安装](./doc/update-install.zh-CN.md)
[English](./doc/update-install.en.md)

auto updater、native installer、npm 迁移、package manager 检测与 rollout 安全。

---

## 仓库结构

### [`npm-original/`](./npm-original/)
原始 npm 产物与关键发布文件。

典型内容包括：

- `anthropic-ai-claude-code-2.1.88.tgz`
- `cli.js`
- `cli.js.map`
- `package.json`
- 提取统计信息

### [`source/`](./source/)
从 `cli.js.map` 恢复出来的源码树。

当前仓库重点保留 Claude Code 自有源码，尽量剥离第三方依赖，方便浏览。

### [`doc/`](./doc/)
研究分析文档目录。

如果你是第一次看这个仓库，这里通常比直接翻源码更值得先看。

### [`README.md`](./README.md) / [`README.zh-CN.md`](./README.zh-CN.md)
英文与中文导航首页。

---

## 文档入口

- [英文主索引](./doc/INDEX.en.md)
- [中文主索引](./doc/INDEX.zh-CN.md)

如果你第一次看这个仓库，我建议：

1. 先读本 README，了解仓库范围
2. 再打开 [中文主索引](./doc/INDEX.zh-CN.md) 看结构化导航
3. 然后按主题选择阅读路径

---

## 推荐阅读顺序

如果你想走一条最直接的主线，建议按下面顺序读：

1. [本 README](./README.zh-CN.md)
2. [文档索引](./doc/INDEX.zh-CN.md)
3. [认证登录](./doc/authentication-login.zh-CN.md)
4. [权限风控](./doc/permissions-risk-control.zh-CN.md)
5. [多 Agent](./doc/multi-agent.zh-CN.md)
6. [MCP](./doc/mcp.zh-CN.md)
7. [远程 / 桥接](./doc/remote-bridge.zh-CN.md)
8. [埋点 / Telemetry](./doc/telemetry.zh-CN.md)
9. [更新 / 安装](./doc/update-install.zh-CN.md)

原因大致是：

- 认证解释身份状态
- 权限解释执行控制
- 多 Agent 与 MCP 解释扩展与协作
- 远程 / 桥接解释高权限远程执行模型
- telemetry 解释可观测性与 rollout 控制
- 更新 / 安装解释客户端交付与迁移策略

---

## 模块分析导航

### 1. 认证登录 / Authentication & Login
- [中文](./doc/authentication-login.zh-CN.md)
- [English](./doc/authentication-login.en.md)

重点：
- OAuth + PKCE
- Claude.ai / Console 双入口
- token 来源管理
- managed auth context
- trusted device 联动

### 2. 权限风控 / Permissions & Risk Control
- [中文](./doc/permissions-risk-control.zh-CN.md)
- [English](./doc/permissions-risk-control.en.md)

重点：
- permission modes
- 规则系统
- classifier 驱动审批
- managed policy
- Bash 安全分析

### 3. 多 Agent / Multi-Agent
- [中文](./doc/multi-agent.zh-CN.md)
- [English](./doc/multi-agent.en.md)

重点：
- AgentTool
- fork subagent
- teammate / swarm runtime
- task list
- mailbox 与 leader approval bridge

### 4. MCP
- [中文](./doc/mcp.zh-CN.md)
- [English](./doc/mcp.en.md)

重点：
- MCP 连接生命周期
- transport 与 auth
- official registry
- 动态 tools / commands / resources
- 受策略控制的扩展模型

### 5. 远程 / 桥接 / Remote / Bridge
- [中文](./doc/remote-bridge.zh-CN.md)
- [English](./doc/remote-bridge.en.md)

重点：
- bridge worker
- session ingress auth
- trusted device
- direct connect
- 远程权限回流路径

### 6. 埋点 / Telemetry
- [中文](./doc/telemetry.zh-CN.md)
- [English](./doc/telemetry.en.md)

重点：
- analytics 入口
- Datadog 与 1P event logging
- GrowthBook
- metadata enrich
- privacy 与 kill switch

### 7. 更新 / 安装 / Update / Install
- [中文](./doc/update-install.zh-CN.md)
- [English](./doc/update-install.en.md)

重点：
- auto updater
- native installer
- npm global/local 迁移
- package manager 检测
- 版本 gate 与 rollout 安全

---

## 按主题快速结论

### 身份体系
Claude Code 把认证当成控制平面的一部分，而不只是“有没有 token”。

### 执行安全
权限系统是分层设计：模式、规则、分类器、shell 级风险分析一起决定是否允许执行。

### 协作运行时
多 Agent 是一级能力，不是附属插件；它有 task、mailbox、leader 控制与多后端执行模型。

### 扩展能力
MCP 深度嵌入命令、工具、权限与 UI，不是一个临时外挂接口。

### 远程执行
远程能力围绕 session 生命周期设计，并且和强身份、可信设备、权限回流紧密绑定。

### 可观测性
telemetry 不是单一日志出口，而是分成 Datadog、1P、OTel 等多层路径，并且带隐私边界控制。

### 交付与升级
安装/更新系统明显处于从 npm 分发向 native installer 分发迁移的过程中。

---

## 方法说明

这个仓库大致按下面步骤构建：

1. 下载 `@anthropic-ai/claude-code@2.1.88` npm 包
2. 解压包内容
3. 检查 `cli.js.map`
4. 从其中内嵌的 `sourcesContent` 恢复文件
5. 剥离/区分第一方源码与第三方打包代码
6. 按模块写结构化分析文档

需要注意：

- 这里的源码来自发布构建，不是原始开发仓库
- 文件名和内容仍然高度有价值，但可能带有构建期变换
- 某些内部专用或 feature gate 逻辑，在公开包里可能是 stub、缺失或部分呈现

---

## 适用范围与局限

这个仓库适合用于：

- 架构研究
- 逆向分析
- 安全审查切入点整理
- 产品能力分析
- 实现模式对比

这个仓库**不等于**：

- 官方源码发布
- 完整无缺的真相
- 与 Anthropic 内部构建完全一致
- 对运行时行为测试的替代品

---

## 语言

- [English README](./README.md)
- [中文 README](./README.zh-CN.md)

---

## 当前状态

已完成模块分析：

- [认证登录](./doc/authentication-login.zh-CN.md)
- [权限风控](./doc/permissions-risk-control.zh-CN.md)
- [多 Agent](./doc/multi-agent.zh-CN.md)
- [MCP](./doc/mcp.zh-CN.md)
- [远程 / 桥接](./doc/remote-bridge.zh-CN.md)
- [埋点 / Telemetry](./doc/telemetry.zh-CN.md)
- [更新 / 安装](./doc/update-install.zh-CN.md)

---

## 包信息

- 包名：`@anthropic-ai/claude-code`
- 分析版本：`2.1.88`
- 还原依据：`cli.js.map` 中内嵌的 `sources` + `sourcesContent`

---

## 后续可以继续补的内容

这个仓库后面还可以继续补这些专题：

- hidden commands / feature flags 专题
- 权限绕过链路深挖
- remote session 协议笔记
- telemetry 隐私边界专题
- MCP auth 深挖

---

## 声明

这个仓库用于对公开发布 npm 包产物进行研究、学习和分析，不代表 Anthropic 官方立场，也不构成官方源码发布。
