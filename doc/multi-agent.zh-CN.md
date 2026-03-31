# 多 Agent 模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/tools/AgentTool/forkSubagent.ts`
- `source/src/tools/shared/spawnMultiAgent.ts`
- `source/src/utils/swarm/inProcessRunner.ts`
- `source/src/utils/swarm/leaderPermissionBridge.ts`
- `source/src/utils/teammateMailbox.ts`
- `source/src/utils/teamDiscovery.ts`
- `source/src/utils/tasks.ts`

## 总体结论

Claude Code 的多 Agent 不是“再开一个子进程”那么简单，而是一整套协作运行时。

从源码看，它至少支持这些协作形态：

1. **单次 subagent / AgentTool 委派**
2. **后台异步 agent**
3. **team / teammate / swarm** 协作
4. **in-process teammate**（同进程上下文隔离）
5. **tmux / pane backend** 这类多窗格 agent
6. **worktree 隔离**
7. **remote 隔离 / remote task**
8. **fork subagent**（继承父上下文的隐式分叉）
9. **任务列表 + mailbox 消息系统 + leader 权限桥**

换句话说，它不是“支持多 agent”，而是把**多 agent 协作**做成了 Claude Code 的核心运行模型之一。

---

## 1. AgentTool 是总入口：`AgentTool.tsx`

### 从 schema 就能看出的能力

`AgentTool` 的输入参数已经暴露出很多设计：

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

这意味着 agent 启动时可以指定：

- 做什么任务
- 用什么类型的 agent
- 用什么模型
- 是否后台运行
- 是否加入某个 team
- 用什么权限模式
- 用什么隔离方式
- 在哪个目录运行

### 输出也不是单一路径

输出 schema 里至少存在这些状态：

- `completed`
- `async_launched`
- 内部还存在：
  - `teammate_spawned`
  - `remote_launched`

也就是说 AgentTool 的调用结果有多种分支：

- 直接同步完成
- 异步后台任务
- 作为 teammate 被拉起
- 远程运行

### 结论

AgentTool 更像一个**委派控制平面**，不是一个普通工具函数。

---

## 2. 多 Agent 不只是实验功能，而是系统级设计

从 `AgentTool.tsx` 的 import 和依赖能看出，多 agent 已经深度嵌入这些系统：

- analytics / growthbook
- LocalAgentTask / RemoteAgentTask
- worktree
- remote teleport
- permissions
- task output storage
- sdk event queue
- agent summary
- MCP 依赖过滤
- agent swarms

这说明多 agent 不是边缘插件，而是横跨：

- UI
- 工具层
- 任务系统
- 权限系统
- 远程系统
- 遥测系统

---

## 3. fork subagent：上下文继承型分叉

`forkSubagent.ts` 很关键，因为它揭示了一个和普通 subagent 不一样的模型：

### 核心特点

当 fork 功能开启时：

- `subagent_type` 可以省略
- 省略时不再走普通 specialized agent，而是触发 **implicit fork**
- 子 agent 会继承：
  - 父对话上下文
  - 父 system prompt
  - 父工具池（精确工具集合）
  - 父模型（`model: inherit`）
- 权限模式是：`bubble`
  - 也就是把权限提示冒泡回父终端

### prompt cache 优化非常明显

`buildForkedMessages()` 的实现非常讲究：

- 保留完整父 assistant message
- 为所有 tool_use block 构造统一的 placeholder `tool_result`
- 只有最后一小段 directive 是每个子任务不同的

这意味着它刻意让多个 fork child 的请求前缀保持**字节级一致**，从而最大化 prompt cache 命中。

### 反递归保护

`isInForkChild()` 会检测 fork boilerplate tag，防止 fork child 再 fork。

### 结论

fork 不是随手拼出来的实验功能，而是做过：

- cache 设计
- 递归保护
- 权限冒泡
- 系统 prompt 继承

的一套成熟分叉机制。

---

## 4. in-process teammate：同进程里的“隔离 agent”

`utils/swarm/inProcessRunner.ts` 是整个多 agent 系统里最有意思的部分之一。

### 它做了什么

注释里写得很清楚，它给 in-process teammate 提供：

- `AsyncLocalStorage` 风格的上下文隔离
- 进度追踪
- AppState 更新
- 完成后的 idle 通知
- plan mode 审批流
- 中止和清理

### 这意味着什么

不是所有 agent 都必须以新进程 / tmux pane 的方式运行。

系统支持：

- **在同一进程里模拟出多个 teammate 的隔离执行上下文**

这通常意味着：

- 更低启动成本
- 更快的任务切换
- 更容易共享某些内存态
- 但也需要非常小心上下文串线问题

所以这里用了 `runWithTeammateContext()`、agent context、独立 mailbox / 任务状态等机制来防串。

### 结论

Claude Code 的多 agent 不是“多进程唯一方案”，而是同时支持：

- 进程外 teammate
- 同进程 teammate

这是一种更高阶的 orchestration 设计。

---

## 5. leader permission bridge：主 Agent 替 worker 审批

`leaderPermissionBridge.ts` 很短，但重要性很高。

它做的事情是：

- 把 leader REPL 的 `ToolUseConfirm` 队列 setter 暴露给非 React 代码
- 把 leader 的 `toolPermissionContext` setter 暴露出去
- 这样 in-process teammate 申请权限时，可以复用 leader 的标准审批 UI

### 在 `inProcessRunner.ts` 里具体体现为

当 teammate 调工具时：

- 如果已有明确 allow/deny，直接走
- 如果是 `ask`：
  - 先尝试 classifier auto-approval（比如 bash）
  - 如果还需要人工审批
  - 就通过 leader 的 ToolUseConfirm dialog 进行确认

### 这意味着什么

多 agent 不是每个 worker 各弹各的权限框，而是：

- **worker 干活**
- **leader 统一审批**

这是一种非常符合人类协作习惯的设计：

- 保持一个主控制面
- 防止多个 agent 各自弹窗把用户淹没

---

## 6. mailbox：agent 之间的文件型通信层

`teammateMailbox.ts` 说明 swarm 并不完全依赖内存或 websocket，而是有一层基于文件的消息系统。

### 结构

每个 teammate 都有 inbox 文件：

- `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

其他 agent 可以往里写消息，接收方再读。

### 特点

- 基于文件
- 有 lockfile 防并发写入
- 记录：
  - from
  - text
  - timestamp
  - read
  - color
  - summary

### 设计意义

这说明 swarm 的通信不是完全耦合在 UI 会话里，而是有一层：

- 可落盘
- 可恢复
- 可跨进程
- 可跨 backend

的轻量通信协议。

这对于：

- tmux teammate
- in-process teammate
- leader / worker 混合模式

都很重要。

---

## 7. 任务系统不是附属品，而是协作骨架：`tasks.ts`

这个文件说明多 agent 协作的另一条主线是 **task list**。

### 它支持什么

- task list ID
- pending / in_progress / completed 状态
- block / blockedBy 依赖关系
- 高水位 task ID
- 并发锁
- team 级 task list
- session fallback

### 一个关键点

`getTaskListId()` 的优先级说明得很清楚：

1. 显式 task list ID
2. in-process teammate 用 leader 的 team name
3. 进程型 teammate 用 `CLAUDE_CODE_TEAM_NAME`
4. leader 创建 team 后使用 team name
5. 否则 fallback 到 session ID

### 这意味着什么

无论 agent 是：

- in-process
- tmux/pane
- standalone

都尽量汇聚到**同一个任务视图**。

也就是说，多 agent 协作不是“消息驱动 alone”，而是：

- mailbox 负责交流
- task list 负责共享任务状态

这两者一起构成协作骨架。

---

## 8. spawnMultiAgent：后端选择与运行形态切换

`spawnMultiAgent.ts` 是真正把 teammate 拉起来的关键拼装模块。

### 它体现出的能力

- 解析 model 继承
- 传播 CLI flags
- 传播 settings/plugin/chrome/model 等运行参数
- 识别 backend：
  - tmux
  - iTerm2 / pane backend
  - in-process backend
- 建立 swarm session
- 写 team file
- 分配 teammate color
- 建 pane / splitpane
- 写 mailbox
- 注册 task

### 特别值得注意的点

#### 权限模式会继承/传播

比如：

- `bypassPermissions`
- `acceptEdits`
- `auto`

都会通过 CLI flag / spawn config 传播到 teammate。

但如果 `planModeRequired` 为 true，就不会继承 bypass 权限。

这说明多 agent 系统和权限系统之间是强耦合的，不是后面补的。

#### model 有 `inherit`

`resolveTeammateModel()` 支持 `inherit`，说明多 agent 很强调和 leader 保持模型一致性，避免上下文能力差太多。

### 结论

spawn 层不是“exec 一个命令”而已，而是一个真正的运行时适配层。

---

## 9. teamDiscovery：说明 swarm 是长期存在的实体

`teamDiscovery.ts` 会读取 team file，产出：

- team summary
- teammate status
- running / idle 状态
- color
- cwd
- worktreePath
- backendType
- mode

### 这意味着什么

team 不是一次性 spawn 完就结束，而是一个可被观察、可被恢复、可被展示的长期结构。

这和普通“subprocess delegate once”完全不是一个级别。

---

## 10. 多 Agent 的真实设计图景

综合这些文件，我认为 Claude Code 的多 agent 架构大致是：

### A. 委派入口

- `AgentTool`
- 各种 Team / Task / SendMessage 工具

### B. 运行形态

- 同步 agent
- 异步后台 agent
- fork child
- in-process teammate
- tmux/pane teammate
- remote agent
- worktree-isolated agent

### C. 协作基础设施

- task list
- teammate mailbox
- team file
- status / discovery
- progress tracking

### D. 控制权收敛

- leader approval bridge
- permission bubbling
- shared task state
- shared team identity

这说明 Claude Code 的多 agent 不是“多个 Claude 一起跑”这么粗糙，而是有一整套：

- 组织结构
- 消息机制
- 任务机制
- 权限代理
- UI 展示
- backend 适配

---

## 11. 模块设计判断

### 我认为这个模块最重要的特点有四个

#### 1）把协作当成一级能力，而不是外挂功能

从 AgentTool、tasks、mailbox、team discovery、spawn backend 来看，多 agent 是核心运行模型之一。

#### 2）同一套系统同时支持多种执行后端

- in-process
- tmux / pane
- remote
- worktree isolation
- background task

这说明它不是绑定某个单一实现，而是做成了可切换的 orchestration backend。

#### 3）leader 是控制中心

worker 不直接各管各的，而是通过：

- leader permission bridge
- team task list
- mailbox
- permission bubbling

把控制权尽量收敛到 leader。

#### 4）非常重视缓存、上下文与复用效率

fork path 的 byte-identical prefix、model inherit、exact tool pool、shared task list，都说明作者在认真优化：

- prompt cache
- context consistency
- orchestration overhead

---

## 12. 值得继续深挖的问题

后续如果继续精读，我建议优先追这些：

1. `runAgent.ts` 里真正的 agent 执行主循环
2. RemoteAgentTask 的会话建立与回传协议
3. team file 的完整 schema 与持久化策略
4. SendMessage / TeamCreate / TaskCreate 系列工具如何形成更高层工作流
5. in-process teammate 的上下文隔离是否完全避免共享状态污染
6. fork child 的权限冒泡在复杂嵌套场景是否有边界问题
7. worktree agent 的提交/合并策略如何和 leader 工作区对齐

---

## 一句话总结

**Claude Code 的多 Agent 模块本质上是一套协作操作系统：它把委派、隔离、通信、任务编排、权限代理和状态展示整合成了一个统一的 swarm/runtime。**
