# MCP 模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/services/mcp/MCPConnectionManager.tsx`
- `source/src/services/mcp/useManageMCPConnections.ts`
- `source/src/commands/mcp/mcp.tsx`
- `source/src/services/mcp/auth.ts`
- `source/src/services/mcp/mcpStringUtils.ts`
- `source/src/services/mcp/officialRegistry.ts`
- `source/src/utils/mcpWebSocketTransport.ts`
- `source/src/components/mcp/MCPSettings.tsx`
- `source/src/components/mcp/MCPReconnect.tsx`

## 总体结论

Claude Code 对 MCP 的支持不是“能连一个外部工具 server”这么简单，而是一套非常深的集成层，至少包括：

1. **连接生命周期管理**
2. **多传输支持**（stdio / SSE / HTTP / WebSocket / Claude.ai proxy）
3. **OAuth / OIDC / token refresh / secure storage**
4. **MCP 工具、命令、资源的统一装载**
5. **配置、启停、重连、认证状态 UI**
6. **策略过滤、官方 registry、channel / permission relay**
7. **与 Agent、权限系统、插件系统、远程 session 的联动**

也就是说，MCP 在 Claude Code 里不是边缘插件接口，而是核心扩展总线之一。

---

## 1. MCPConnectionManager：连接控制面

`MCPConnectionManager.tsx` 本身不复杂，但它定义了 MCP 的控制面抽象：

- `reconnectMcpServer(serverName)`
- `toggleMcpServer(serverName)`

然后通过 React context 暴露给 UI / 命令系统。

### 这说明什么

MCP 不是“初始化时连一下就算了”，而是运行期可控的：

- 可重连
- 可启停
- 可被设置界面和 `/mcp` 命令直接操作

这也是后面 `MCPReconnect.tsx` 和 `mcp.tsx` 能工作的基础。

---

## 2. 真正的重头戏在 `useManageMCPConnections.ts`

这个 hook 几乎就是 MCP 运行时管理器。

### 它处理的事情包括

- 初始化 MCP 连接
- 跟 AppState 同步
- 自动重连
- 工具 / 命令 / 资源的动态更新
- plugin MCP server 刷新
- Claude.ai MCP 配置拉取
- enterprise policy / config filtering
- notification / error 去重
- channel permission relay
- elicitation handler 注册
- 批量更新 AppState

### 一些特别关键的点

#### 1）MCP 输出不止工具

它会同时维护：

- `mcp.clients`
- `mcp.tools`
- `mcp.commands`
- `mcp.resources`

所以在 Claude Code 里，MCP 不是单纯“提供 tools”，而是完整的协议实体：

- tools
- commands
- resources

#### 2）连接是动态的

代码里有：

- reconnect backoff
- connection lifecycle handler
- list changed notifications
- resource/tool/command 增量刷新

说明它把 MCP server 当成**持续在线的能力源**，不是一次性扫描。

#### 3）会受 policy 和配置过滤

你能直接看到这些逻辑参与了 MCP：

- `filterMcpServersByPolicy`
- `doesEnterpriseMcpConfigExist`
- `isMcpServerDisabled`
- `setMcpServerEnabled`
- `dedupClaudeAiMcpServers`

也就是说 MCP 并不是“用户填个地址就全开”，而是：

- 有策略层
- 有企业配置层
- 有 enabled/disabled 状态
- 有重复源去重逻辑

#### 4）和 channel / relay 有耦合

文件里还出现了：

- channel notification
- channel permission relay
- `registerElicitationHandler`

这说明某些 MCP server 可能不仅是普通本地工具，还可能与交互式审批、消息通道、外部 relay 绑定。

### 结论

`useManageMCPConnections.ts` 说明 Claude Code 是把 MCP 当成一个**可热插拔、可策略控制、可持续同步**的运行时能力网络来管理的。

---

## 3. `/mcp` 命令：不是只读页面，而是运维入口

`commands/mcp/mcp.tsx` 暴露了这些操作：

- `/mcp`
- `/mcp reconnect <server>`
- `/mcp enable <server|all>`
- `/mcp disable <server|all>`
- `/mcp no-redirect`

### 一个细节

它内部专门做了 `MCPToggle` 组件，只为了在组件上下文里拿到 `toggleMcpServer()`。

这说明当前 MCP 控制逻辑还是明显偏 React / UI context 驱动，而不是完全独立的 service API。

### 另一个有意思的点

对某些构建/用户类型，基础 `/mcp` 命令会重定向到 plugin 管理视图。

这说明产品层面已经把 MCP 和 plugin 管理体验部分融合了。

### 结论

`/mcp` 更像一个运维入口，而不是“列一下配置”的静态命令。

---

## 4. MCPSettings：MCP 不只是后台状态，还有完整的管理 UI

`components/mcp/MCPSettings.tsx` 让 MCP 的产品形态更清楚了。

### 它做了什么

- 从 AppState 读取当前 `mcp.clients`
- 读取 agent definitions，再提取 agent 依赖的 MCP server
- 识别不同 transport：
  - `stdio`
  - `sse`
  - `http`
  - `claudeai-proxy`
- 对 SSE / HTTP server 检查认证状态
- 根据 server 类型切换不同菜单 / 工具视图
- 在没有配置时提示 `/doctor` 或官方文档

### 这说明什么

MCP 在 UI 上已经是一个完整的“server inventory + capability browser”：

- server 列表
- transport 类型
- auth 状态
- 工具列表
- agent 依赖 server
- 远程代理 server

它不是藏在配置文件里的黑盒功能。

---

## 5. MCPReconnect：状态机很直接

`MCPReconnect.tsx` 虽然不长，但能反映 MCP client 的状态模型。

重连结果至少会区分：

- `connected`
- `needs-auth`
- `pending`
- `failed`
- `disabled`

### 这说明什么

Claude Code 对 MCP server 的认知不是简单的 online/offline，而是更接近：

- 已连接
- 等认证
- 正在连
- 失败
- 被禁用

这也和前面的控制面、设置 UI、enable/disable 命令相互对应。

---

## 6. MCP Auth：这块非常重，说明远不止“拿个 token”

`services/mcp/auth.ts` 是整个 MCP 模块里最复杂的文件之一。

### 一眼可见的能力

它直接依赖了 `@modelcontextprotocol/sdk` 的 auth 能力：

- auth discovery
- OAuth server discovery
- auth metadata
- token refresh
- OAuth client information
- OAuth token schema

同时还额外自己做了很多事情：

- localhost callback server
- PKCE / state / nonce 相关处理
- secure storage
- lockfile
- timeout / retry
- XAA（cross-app access）
- IdP/OIDC 发现与登录
- xss 过滤
- logging redaction
- OAuth 错误标准化

### 关键观察 1：它非常在意 OAuth server 的不规范行为

代码里专门处理了：

- 某些 OAuth 服务器返回 HTTP 200 但 body 里是 error
- Slack 风格的非标准 `invalid_refresh_token` / `expired_refresh_token` / `token_expired`
- 把它们归一化成 `invalid_grant`

这说明作者不是只按 RFC 理想路径写的，而是踩过真实世界各种 provider 的坑。

### 关键观察 2：非常强调敏感参数脱敏

它显式 redaction：

- `state`
- `nonce`
- `code_challenge`
- `code_verifier`
- `code`

这说明 MCP OAuth 流程不只是“能通”，还对日志安全很敏感。

### 关键观察 3：会结合 secure storage 和本地锁

这表明：

- token 不是纯内存态
- 需要持久化
- 需要并发安全
- 多进程 / 多会话场景下可能同时读写

### 关键观察 4：MCP auth 不只是 OAuth，还带企业身份/跨应用接入能力

你能看到：

- `performCrossAppAccess`
- `discoverOidc`
- `acquireIdpIdToken`
- `getXaaIdpSettings`
- `isXaaEnabled`

这说明 MCP 连接的认证模型已经延伸到：

- OIDC
- IdP
- 企业/跨应用接入

### 结论

MCP auth 是一个独立复杂系统，不是给外部 server 随手补个 login 按钮。

---

## 7. officialRegistry：官方生态入口

`officialRegistry.ts` 会拉取：

- `https://api.anthropic.com/mcp-registry/...`

并把官方 MCP URL 缓存在内存中，用于：

- 判断某个 MCP URL 是否属于官方 registry

### 这说明什么

Claude Code 不只是“兼容 MCP 协议”，还在产品层准备了：

- 官方 MCP 生态
- 商业可见性 registry
- 官方/非官方来源区分

这很像一个生态平台的雏形。

---

## 8. `mcpStringUtils.ts`：MCP 已深度嵌入权限和命名系统

这里有几个很重要的工具：

- `mcpInfoFromString`
- `getMcpPrefix`
- `buildMcpToolName`
- `getToolNameForPermissionCheck`
- `getMcpDisplayName`

### 为什么重要

它说明 MCP 工具在 Claude Code 内部不是“外挂别名”，而是被纳入统一命名体系：

- `mcp__server__tool`

这样做的目的非常明确：

- 防止 MCP 工具名和内建工具名冲突
- 让权限规则可以精确作用到：
  - 某个 MCP server
  - 某个 MCP tool

### 结论

MCP 在权限系统里是一级对象，而不是 UI 层展示名。

---

## 9. WebSocket transport：MCP 连接不是只靠 stdio/SSE

`mcpWebSocketTransport.ts` 说明它还实现了 JSON-RPC over WebSocket transport。

### 特点

- 同时兼容：
  - Bun 原生 WebSocket
  - Node `ws`
- 统一封装为 MCP SDK 所需的 `Transport`
- 做 JSON parse / schema parse
- 对 message / error / close 做统一处理
- 会在错误时打诊断日志

### 这说明什么

Claude Code 并没有把 MCP transport 限死在早期常见的 stdio / SSE，而是在往更广义的远程 transport 演化。

这也能和 remote / bridge 模块形成潜在联动。

---

## 10. MCP 在 Claude Code 中的真实位置

综合这些文件，我认为 MCP 在 Claude Code 中扮演的是：

### A. 扩展能力总线

MCP server 可以向运行时注入：

- tools
- commands
- resources

### B. 受控连接层

通过：

- config
- enable / disable
- reconnect
- auth state
- policy filtering
- official registry

来控制这些能力是否真正进入运行时。

### C. 与其他子系统的桥梁

MCP 不是孤岛，它明显和这些系统耦合：

- 权限系统
- agent / team / agent definitions
- plugin 管理
- channel relay
- remote session / ingress auth
- telemetry / analytics

### D. 未来生态入口

官方 registry、Claude.ai proxy、enterprise config、OAuth/OIDC 都说明 MCP 不只是兼容协议，而是在往“受管生态平台”方向走。

---

## 11. 模块设计判断

### 我认为这个模块最重要的特点有四个

#### 1）MCP 是一级扩展模型，不是外挂脚本接口

工具、命令、资源都被统一纳入 AppState 和 UI 体系。

#### 2）MCP auth 非常重，已经接近独立产品模块

OAuth、OIDC、IdP、XAA、secure storage、error normalization，都说明它面对的是复杂真实环境。

#### 3）MCP 受策略和产品化管理约束

并不是用户随便加个 server 就算结束，而是有：

- enterprise policy
- enable/disable
- auth gating
- official registry
- config dedup / filtering

#### 4）MCP 深度参与 Claude Code 的能力拼装

它直接影响：

- agent 能用哪些工具
- 权限规则如何命名和匹配
- 命令系统能出现哪些动态命令
- 资源浏览和交互方式

---

## 12. 值得继续深挖的问题

后续如果继续精读，我建议优先追这些：

1. `client.ts` 里 MCP client 创建与 reconnect 细节
2. `config.ts` 里本地、项目、企业、Claude.ai 的配置合并规则
3. Claude.ai proxy server 的具体语义和鉴权链路
4. `registerElicitationHandler()` 如何与交互审批集成
5. channel permission relay 的使用场景
6. 远程 session ingress token 与 MCP auth 的边界
7. resource / prompt / tool list changed notification 的完整刷新路径

---

## 一句话总结

**Claude Code 的 MCP 模块本质上是一层受控扩展总线：它把外部能力源接入到工具、命令、资源、权限、认证和 UI 体系里，并且对其进行持续管理、策略约束与产品化包装。**
