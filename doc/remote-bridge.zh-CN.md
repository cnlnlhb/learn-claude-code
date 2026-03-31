# 远程 / 桥接模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/bridge/bridgeMain.ts`
- `source/src/bridge/trustedDevice.ts`
- `source/src/remote/RemoteSessionManager.ts`
- `source/src/remote/SessionsWebSocket.ts`
- `source/src/server/createDirectConnectSession.ts`
- `source/src/server/directConnectManager.ts`
- `source/src/utils/sessionIngressAuth.ts`
- `source/src/utils/teleport/api.ts`

## 总体结论

Claude Code 的远程 / 桥接能力不是“连到远程机器跑一下”那么简单，而是一整套会话接入体系，至少包含：

1. **桥接 worker / bridge poll loop**
2. **远程 session WebSocket 订阅**
3. **direct connect 模式**
4. **session ingress token 注入与轮换**
5. **trusted device / elevated auth**
6. **remote permission request / control channel**
7. **teleport 风格的远程事件投递与 Session API**
8. **worktree / 多 session / remote task 与 bridge 的联动**

换句话说，它不是“一个 remote shell”，而是一个带认证、会话、权限控制、重连、观察性和多形态 transport 的远程执行平面。

---

## 1. `bridgeMain.ts`：远程桥接主循环是一个真正的 daemon 运行时

`bridgeMain.ts` 是整个远程能力里最重的文件之一。

### 从 imports 和状态变量就能看出的事情

它直接耦合了：

- analytics / GrowthBook
- datadog / event logging shutdown
- bridge API client
- token refresh scheduler
- session spawner
- trusted device token
- worktree
- multi-session spawn
- capacity wake
- environment/session id 兼容层
- worker registration / work secret

### `runBridgeLoop()` 的职责

从函数签名和内部状态能看到，它至少维护：

- `activeSessions`
- `sessionStartTimes`
- `sessionWorkIds`
- `sessionCompatIds`
- `sessionIngressTokens`
- `sessionTimers`
- `completedWorkIds`
- `sessionWorktrees`
- `timedOutSessions`
- `titledSessions`

这说明 bridge 不是单纯“拉到一条任务就开跑”，而是在管理一个持续的远程 session fleet。

### 一个重要判断

bridge worker 的定位更像：

- **远程控制面上的执行代理**
- 而不是一次性 CLI 命令

因为它要做的事情包括：

- 轮询工作
- 维护 session 生命周期
- 为 active session 发 heartbeat
- token 过期后 re-queue/reconnect
- capacity 满时休眠并在空位释放后唤醒
- 为每个 session 跟踪 worktree / timer / title / auth token

### 结论

`bridgeMain.ts` 表明 Claude Code 的 bridge 是一个真正的长生命周期 worker runtime。

---

## 2. bridge 把“会话”当成中心对象，而不是命令

在 `bridgeMain.ts` 里，最核心的不是单条命令执行，而是 session lifecycle。

### 一些明显信号

- `SessionSpawner`
- `SessionHandle`
- `SessionDoneStatus`
- `initialSessionId`
- `sessionIdCompat`
- `reconnectSession`
- session timeout watchdog
- session title management

### 这意味着什么

Claude Code 的远程模型不是：

- 发一个请求
- 远端返回一个结果

而是：

- 创建一个会话
- 绑定身份与 token
- 让这个会话持续接收输入和输出
- 允许重连、恢复、命名、超时治理

这也是它能支持 remote assistant / remote task / remote agent 的基础。

---

## 3. trusted device：远程控制链路里的“增强认证”

`bridge/trustedDevice.ts` 非常关键，因为它直接说明 bridge session 不是普通权限等级。

源码注释里写得很清楚：

- bridge sessions 在服务端是 `SecurityTier=ELEVATED`
- CLI 侧是否发送 `X-Trusted-Device-Token` 由 gate 控制
- trusted device token 持久化到 keychain
- enrollment 必须在 `/login` 后很短时间内完成

### 关键机制

- `getTrustedDeviceToken()`：读取并按 gate 决定是否发送
- `clearTrustedDeviceToken()`：登录时先清理旧 token，防止跨账号串用
- `enrollTrustedDevice()`：调用 `/auth/trusted_devices` 为本机登记 device token

### 这说明什么

Claude Code 的远程/桥接能力在身份模型里属于更高敏感度操作。

它不满足于：

- 有 OAuth token 就行

而是额外引入：

- trusted device
- keychain persist
- account switch 时清旧 token
- freshness constraints（登录后短窗口内 enrollment）

### 结论

trusted device 说明 bridge 不是普通 API 功能，而是更接近“把这台机器登记为可信远程端点”。

---

## 4. RemoteSessionManager：远程会话不仅有消息流，还有控制流

`remote/RemoteSessionManager.ts` 很能体现远程会话的产品形态。

### 它管理什么

- WebSocket 订阅远程 session 输出
- HTTP POST 发送用户消息到远程 session
- permission request / response
- cancel request
- reconnect / disconnect 回调

### 最重要的一点

远程消息不是只有 assistant text。它区分：

- `SDKMessage`
- `control_request`
- `control_response`
- `control_cancel_request`

而且 `control_request` 里重点支持：

- `can_use_tool`

### 这意味着什么

远程 agent 在执行工具之前，依然可以把权限请求抛回本地客户端审批。

也就是说，远程执行不是“权限都在远端独立决定”，而是支持：

- **远程执行 + 本地审批回路**

这点非常重要，因为它把远程执行和权限模型打通了。

---

## 5. SessionsWebSocket：远程 session 是可重连的长期订阅

`remote/SessionsWebSocket.ts` 进一步说明远程 session 通道的实现细节。

### 关键特征

- 连接到 `/v1/sessions/ws/{sessionId}/subscribe`
- 用 Authorization header 做认证
- 支持 Bun 和 Node `ws`
- 有 ping/pong keepalive
- 有 reconnect 逻辑
- 区分永久 close code 和临时异常
- 对 `4001 session not found` 有有限次重试

### 这说明什么

远程 session 不是“偶尔开一下 socket”，而是按长期订阅设计的：

- 有保活
- 有重连
- 有 close code 语义
- 会考虑 compaction 等短暂 session 状态抖动

这已经是成熟实时客户端的写法了。

---

## 6. session ingress auth：token 注入链路很讲究

`utils/sessionIngressAuth.ts` 很值得看，因为它解释了远程会话 token 怎么进到当前进程里。

### token 来源优先级

1. `CLAUDE_CODE_SESSION_ACCESS_TOKEN` 环境变量
2. `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` 文件描述符
3. well-known file（例如 `.session_ingress_token`）

### 设计意义

这说明远程 token 注入考虑了很多运行场景：

- 父进程直接注入 env
- 通过 FD 安全传 token
- 子进程无法继承 FD 时，从 well-known file 兜底

### 还有两个关键点

#### 1）token 是会被 in-process 更新的

`updateSessionIngressAuthToken()` 会直接更新 env var，说明桥接重连后可以给当前进程热注入新 token，而不需要重启。

#### 2）session key 和 JWT 走不同 header

如果 token 以 `sk-ant-sid` 开头，会走：

- Cookie `sessionKey=...`
- `X-Organization-Uuid`

否则走：

- `Authorization: Bearer ...`

### 结论

session ingress auth 不是一个简单字符串，而是一套跨进程、可轮换、兼容多种身份形态的注入链路。

---

## 7. direct connect：除了云端会话，还有直连服务器模式

`server/createDirectConnectSession.ts` 和 `server/directConnectManager.ts` 说明还有另一条路径：

- 直接连某个 server URL
- `POST /sessions`
- 返回 sessionId / wsUrl / workDir
- 然后在 WebSocket 上用流式 JSON 和 control message 交互

### direct connect 的特点

- 不一定依赖完整 Sessions API / bridge 云端路径
- 更像一个“直接对接 remote code server”的模式
- 同样支持：
  - message 发送
  - permission request
  - interrupt
  - control response

### 这意味着什么

Claude Code 的远程执行平面不是单一架构，而是至少有：

- bridge / CCR / Sessions API 模式
- direct connect 模式

也就是：

- **托管远程会话**
- **直连远程端点**

两条路并存。

---

## 8. teleport API：远程能力和 Session API / 仓库上下文深度耦合

`utils/teleport/api.ts` 让远程体系的产品边界更清楚了。

### 它处理的内容包括

- Session API 请求重试
- CodeSession 列表与解析
- org UUID 获取
- Claude.ai OAuth token 作为前置条件
- repository / git source / knowledge base source
- session context
- outcomes / branches / GitHub PR
- BYOC beta 标记

### 这说明什么

Claude Code 的 remote session 不是纯粹“给一个 shell”，而是和这些更高层实体绑定：

- 仓库
- Git revision
- knowledge base
- outcome branches
- session context
- remote code session

也就是说，远程执行被产品化成了“代码会话”而不仅是“远程终端”。

---

## 9. 远程 / 桥接与其他系统的耦合

综合这些文件，远程 / 桥接能力明显与这些系统强耦合：

- **认证系统**
  - OAuth token
  - org UUID
  - trusted device
  - session ingress token

- **权限系统**
  - remote control request
  - `can_use_tool` permission flow
  - 本地审批 / 远端执行

- **MCP / agent / session**
  - 远程 session 也可以承载动态工具能力
  - bridge 管理多个 session / remote tasks

- **多 agent / worktree**
  - bridge 可以配合 session spawner / worktree spawn

- **遥测系统**
  - bridge lifecycle、heartbeat error、token failure、shutdown 等都打点

### 结论

远程 / 桥接不是单独一块网络代码，而是 Claude Code 里几乎所有“高权限运行时能力”的交汇点之一。

---

## 10. 模块设计判断

### 我认为这个模块最重要的特点有四个

#### 1）远程能力以 session 为中心，而不是命令为中心

无论 bridge、remote websocket、teleport、direct connect，都在围绕 session lifecycle 设计。

#### 2）本地客户端仍然保有控制权

即使在远程执行场景下，permission request 也可以回流到本地审批。

#### 3）身份和设备信任是这个模块的核心

trusted device、session ingress token、org UUID、OAuth token 都说明：远程桥接不是普通低权限 API，而是受严格身份模型保护的能力。

#### 4）它已经具备平台化雏形

bridge worker、Session API、BYOC、direct connect、remote session viewer、multi-session spawn 都说明这套系统已经不是单点功能，而是一个平台层。

---

## 11. 值得继续深挖的问题

后续如果继续精读，我建议优先追这些：

1. `bridgeApi.ts` 的完整接口与错误语义
2. `sessionRunner.ts` 如何把本地 claude 进程绑到 bridge session
3. direct connect server 侧协议和安全边界
4. trusted device 在服务端如何校验、失效、轮换
5. `sendEventToRemoteSession()` 的完整 HTTP 协议
6. remote session 与 compaction / transcript / task 系统如何交互
7. 多 session / 多环境 gate 在外部版本中到底开到了什么程度

---

## 一句话总结

**Claude Code 的远程 / 桥接模块本质上是一层高权限远程会话平面：它把 session 生命周期、身份认证、可信设备、权限回流、重连机制和多种 transport 组合成了一个可以长期运行的远程执行系统。**
