# 权限风控模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/utils/permissions/PermissionMode.ts`
- `source/src/utils/permissions/permissions.ts`
- `source/src/utils/permissions/yoloClassifier.ts`
- `source/src/utils/permissions/bypassPermissionsKillswitch.ts`
- `source/src/utils/permissions/permissionsLoader.ts`
- `source/src/utils/permissions/denialTracking.ts`
- `source/src/tools/BashTool/bashSecurity.ts`

## 总体结论

Claude Code 的权限风控模块不是一个“弹一次确认框”的简单实现，而是一套分层设计的执行控制系统，大致可以拆成四层：

1. **模式层**：决定当前会话处于什么权限模式
2. **规则层**：根据 allow / deny / ask 规则做静态匹配
3. **分类器层**：用 classifier / auto mode / yolo classifier 对高风险行为再做动态判断
4. **命令安全层**：对 Bash 命令本身做大量语法和绕过检测

它的目标不是单纯“别让用户点错”，而是把工具调用控制做成一个真正的策略系统。

---

## 1. 权限模式：`PermissionMode.ts`

### 可见的模式

从代码里能直接看到这些模式：

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto`（在 `TRANSCRIPT_CLASSIFIER` feature 打开时存在）

这说明系统并不是“允许 / 拒绝”二元模型，而是有多种运行姿态。

### 设计含义

- `default`：正常审批模式
- `plan`：偏规划、暂停执行
- `acceptEdits`：更激进，偏向自动接受编辑
- `bypassPermissions`：几乎等于关闭大部分人工审批
- `dontAsk`：不再询问，风险更高
- `auto`：由分类器和 gate 控制的自动模式

### 一个重要细节

`auto` 对 external users 并不是普通外部模式的一部分，它和内部/feature gate 明显有关。

这说明 Claude Code 的权限模式分成：

- 对外暴露的稳定模式
- 内部或受控 rollout 的附加模式

也就是“权限模式本身就是产品能力分层的一部分”。

---

## 2. 规则系统：`permissions.ts`

这是权限系统的主干文件之一。

### 规则来源

代码里把规则来源抽象成多个 source：

- settings 相关来源
- `cliArg`
- `command`
- `session`

配合 `permissionsLoader.ts`，还能看到磁盘上的设置来源包括：

- `userSettings`
- `projectSettings`
- `localSettings`
- `policySettings`
- 以及一些 flag / managed settings 场景

### 规则行为

主要有三类：

- `allow`
- `deny`
- `ask`

也就是允许、拒绝、必须询问。

### 规则匹配能力

这个模块不只是支持按工具名做简单匹配，还支持：

- 工具级规则
- 带内容的规则（例如 `Tool(content)` 语义）
- Agent 类型规则
- MCP server 级别规则
- MCP tool 级别规则
- 多操作命令拆分后的逐段判断

例如：

- 整个工具是否匹配某条规则
- 某个 agentType 是否被 deny
- MCP 规则是否命中某个 server 下全部工具

### 结论

这已经不是“白名单按钮”，而是一个支持多维匹配对象的策略匹配器。

---

## 3. 提示文案其实暴露了风控模型

`createPermissionRequestMessage()` 很值得看，因为它把内部的决策理由暴露得很清楚。

系统会根据不同 reason type 给出不同来源的审批原因，例如：

- classifier 触发
- hook 阻断
- rule 命中
- 子命令拆分后部分需要审批
- permission prompt tool 要求审批
- sandbox override
- workingDir 原因
- safetyCheck
- mode 限制
- asyncAgent

这说明在它内部，“为什么要询问”不是单一原因，而是多个子系统都能触发 ask：

- 静态规则
- 动态 classifier
- hook 系统
- sandbox
- 工作目录限制
- 权限模式本身

这是一种**可解释的策略系统**，不是纯黑盒。

---

## 4. managed policy 能压过用户本地规则：`permissionsLoader.ts`

这里一个很关键的点是：

`allowManagedPermissionRulesOnly`

如果这个开关打开，系统只会信任：

- `policySettings`

而忽略其他普通来源的规则。

### 这意味着什么

这意味着 Claude Code 支持一种“受管权限策略”模式：

- 用户自己在本地配的 allow/deny/ask 规则
- 可能被组织或策略设置整体压过去

并且 UI 上“always allow”之类的选项还能被隐藏：

- `shouldShowAlwaysAllowOptions()`

### 结论

这不是单机工具的思路，而是明显考虑了：

- 企业受管环境
- 集中策略下发
- 用户自主权限和组织权限的优先级冲突

也就是说，它把权限系统做成了**个人模式 + 托管模式**双轨架构。

---

## 5. bypassPermissions / auto mode 并不是绝对开放：`bypassPermissionsKillswitch.ts`

这个文件很重要，因为它说明：

- `bypassPermissions` 不是永远能开
- `auto mode` 也不是永远能开

### bypassPermissions 的处理

代码会在运行初期做一次 gate 检查：

- 如果当前上下文允许 bypassPermissions
- 但服务端/Statsig gate 判定应该禁用
- 就会把当前 `toolPermissionContext` 替换成一个“disabled bypass”版本

而且登录后会 reset 再重新检查，说明：

- 权限能力和组织身份/账号状态有关

### auto mode 的处理

`checkAndDisableAutoModeIfNeeded()` 会：

- 检查当前模型/fastMode/权限上下文
- 调 `verifyAutoModeGateAccess(...)`
- 如果不满足 gate 条件，就更新 context
- 必要时还会往通知队列塞高优先级 warning

### 结论

即便用户切到了某种“更放权”的模式，系统仍然保留：

- 远程 gate
- 组织级限制
- 动态熔断

也就是说，所谓 bypass / auto，更像“有条件下发的能力”，不是裸开关。

---

## 6. denial tracking：分类器不会无限硬拒绝

`denialTracking.ts` 很短，但很有意思。

它维护两个计数：

- `consecutiveDenials`
- `totalDenials`

阈值是：

- 连续拒绝 >= 3
- 总拒绝 >= 20

达到阈值后：

- `shouldFallbackToPrompting(...)` 返回 true

### 设计含义

这意味着 classifier 不是一条路走到黑。

如果分类器连续拒绝太多，系统会回退到 prompting。

这背后的产品逻辑很明显：

- 如果 classifier 过于保守
- 持续挡住用户工作流
- 系统宁可回退到人工审批，也不继续把用户卡死

这是一个**可用性兜底**机制。

---

## 7. YOLO / auto mode classifier：`yoloClassifier.ts`

这里的命名已经非常直接了：

- yolo classifier
- auto mode system prompt
- external permissions template
- anthropic permissions template

### 可以看出的事实

1. auto mode 背后有一套 classifier prompt
2. 这套 prompt 还分：
   - external 版本
   - anthropic internal 版本
3. 支持把 classifier request / response dump 到本地
4. 遇到错误时还能把系统 prompt、用户 prompt、token 对比信息写到 dump 文件
5. classifier 还统计：
   - 主循环 token
   - classifier token 估算
   - transcript 条目数
   - model
   - action

### 结论

这说明 auto mode 并不是“一个 if 开关”，而是一个真实的模型驱动决策子系统。

它甚至有：

- prompt 模板体系
- debug dump 机制
- context 对比诊断
- feature flag / rollout 控制
- external/internal prompt 差异

这已经接近“二级风险判断模型”了。

---

## 8. Bash 风险检测很重：`tools/BashTool/bashSecurity.ts`

这是整个风控体系里最硬核的部分之一。

### 代码风格一眼能看出的特点

它不是仅靠一个 regex 拦危险命令，而是在做大量针对 shell 绕过技巧的防守。

### 它显式处理的风险点包括

#### 命令替换 / 进程替换 / 参数替换

- `$()`
- `` `...` ``
- `<()`
- `>()`
- `${}`
- `$[]`

#### zsh 特有绕过

- `=cmd` equals expansion
- `always {}`
- zmodload 相关模块能力
- `zpty`
- `ztcp`
- `sysopen / syswrite / sysread`
- `zf_rm / zf_mv ...`

#### heredoc 与 substitution 混合

代码里有专门的 `isSafeHeredoc()` 逻辑，并且注释写得非常谨慎：

- 如果这里 early-allow 判断错了
- 会直接绕过后续所有 validator

所以实现必须是“可证明安全”，而不是“大概安全”。

#### 其他防护点

- malformed token injection
- IFS injection
- brace expansion
- control characters
- unicode whitespace
- backslash escaped operators
- comment quote desync
- quoted newline
- jq system function / file args
- proc environ access
- output/input redirection 相关问题

### 结论

这说明 BashTool 的风控设计假设是：

> 攻击者会利用 shell 解析差异、zsh 特性、重定向、转义、编码、换行、注释不同步等技巧绕过简单规则。

所以它的策略不是“黑名单几个命令”，而是把 shell 作为攻击面来对待。

---

## 9. 一个重要现实：外部构建里有 stub

你给我看到的 `bashClassifier.ts` 这一份是 external build stub：

- `isClassifierPermissionsEnabled() -> false`
- `classifyBashCommand(...)` 固定返回 disabled

这说明：

- 并不是所有构建都会启用完整 classifier 权限能力
- 外部构建和内部构建在权限层面可能有能力差异

这和前面 `USER_TYPE === 'ant'`、feature gate、external/internal prompt 模板是吻合的。

所以看权限系统时，要区分：

- 架构上支持什么
- 当前这个发行包实际启用了什么

---

## 10. 模块设计判断

### 这个模块最核心的设计特点

#### 1）不是单层审批，而是多层防线

从代码看，至少有：

- 模式约束
- 规则匹配
- 分类器判断
- 命令级静态/语法安全分析
- hook
- sandbox
- managed policy

#### 2）不是纯用户本地控制，而是支持组织治理

- managed permission rules only
- gate 控制 bypass / auto
- settings source 优先级

#### 3）不是只防“误点确认”，而是在防策略绕过

尤其 Bash 安全层已经明显在考虑：

- shell parser 差异
- zsh 模块滥用
- heredoc 早放行漏洞
- 注释 / 引号不同步
- 重定向伪装

#### 4）可用性有兜底

- denial tracking 超阈值后回退 prompting

这说明作者不想把系统做成“永远最保守”，而是想在安全和可用性之间维持一个动态平衡。

---

## 11. 值得继续深挖的问题

后续继续精读时，我建议优先追这些：

1. `permissionSetup.ts` 如何创建 disabled / gated context
2. classifierDecision.js 的具体判定路径
3. `shouldUseSandbox()` 与权限规则如何交叉决策
4. MCP 工具在权限体系里的命名与权限边界
5. async agent / agent tool 的审批边界如何定义
6. bashSecurity 最终是如何和 allow/deny/ask 规则合流的
7. external build 与 ant build