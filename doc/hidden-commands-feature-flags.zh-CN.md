# Hidden Commands / Feature Flags 专题

[返回 README](../README.zh-CN.md) · [文档索引](./INDEX.zh-CN.md) · [English Version](./hidden-commands-feature-flags.en.md)

这份文档总结了从 `@anthropic-ai/claude-code@2.1.88` 公开 npm 包中，能够看出的隐藏命令、feature gate，以及公开构建里的 stub 痕迹。

---

## 摘要结论

公开包里很明显能看到一些“像内部命令/调试命令”的痕迹，但这里最重要的模式是：

- **很多可疑命令在公开包里只是 hidden stub**
- 这些 stub 通常长这样：
  - `isEnabled: () => false`
  - `isHidden: true`
  - `name: 'stub'`

这强烈说明：公开构建保留了某些内部/调试命令的**命令位和结构痕迹**，但把真正实现删掉了，或者替换成了不可用占位。

所以最稳妥的结论不是“这些隐藏命令都能用”，而是：

> 公开包仍然泄露了内部/调试能力面的轮廓，即便真实实现已经被裁掉或 gate 掉了。

---

## 1. 这里的“隐藏命令”主要有哪些形态

从恢复出来的包来看，隐藏能力至少有三种形态：

1. **命令文件存在，但实现被 stub 掉**
2. **命令是否启用依赖 build flag / user type / env var**
3. **能力只有在 feature gate 打开时才注册或生效**

这和普通“help 里能看到的公开命令列表”不是一回事。

---

## 2. 在公开包里明确看到的 stub 命令

目前确认到，下面这些路径在公开包里是明显的 stub：

- `source/src/commands/bughunter/index.js`
- `source/src/commands/mock-limits/index.js`
- `source/src/commands/oauth-refresh/index.js`
- `source/src/commands/ctx_viz/index.js`

它们恢复出来的内容几乎是统一模板：

```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

### 这说明什么

这意味着：

- 公开构建里仍然保留了这些命令名/模块位
- 但真实逻辑已经被删掉或替换
- 它们默认不会出现在正常命令面板里
- 也不能在公开包里直接按真实功能使用

### 仅从命名能看出的方向

这些名字很像内部工具，例如：

- `bughunter`：故障排查 / bug 复现支持
- `mock-limits`：额度 / quota / rate limit 模拟
- `oauth-refresh`：强制刷新 OAuth token 的调试入口
- `ctx_viz`：上下文可视化 / prompt/context 调试工具

也就是说，即使实现被删了，**命名本身仍然具有结构信息价值**。

---

## 3. 其他可疑命令名与调试痕迹

除了上面的 stub 文件，之前对源码路径和字符串的扫描里，还看到过一些名字，例如：

- `ant-trace`
- `break-cache`
- `perf-issue`
- 某些 install-helper / app-install 类路径

并不是所有这些名字都已经确认在公开 build 中是可执行命令，但命名风格很一致。

### 它们共同透露出的方向

这些名字大多像：

- 内部诊断工具
- 性能调查工具
- cache 清理 / 失效工具
- auth/session 维护工具
- support/运营排障工作流

这和一个成熟内部运维/调试工具链的形态非常吻合。

---

## 4. build-time feature gate 很重

从恢复出来的代码看，大量能力都受 `feature('...')` 控制（来自 `bun:bundle`）。

在 `main.tsx` 和命令相关代码里，明显能看到这些 feature 名：

- `feature('NEW_INIT')`
- `feature('COORDINATOR_MODE')`
- `feature('KAIROS')`
- `feature('TRANSCRIPT_CLASSIFIER')`

### 这意味着什么

Claude Code 不是“一个构建里所有能力都打开”的单体。

而更像：

- 不同 build 存在 dead-code elimination
- 有条件 import
- 某些功能只在特定 build 中出现
- 某些公开包里只留了占位结构

### 很关键的一点

**源码里出现某个文件或能力，不等于它在公开包运行时一定可达。**

这是读这个包时必须一直记住的前提。

---

## 5. `USER_TYPE === 'ant'` 是非常关键的一条边界

在很多地方，代码都会显式区分：

- Anthropic 内部用户
- 外部公开用户

这在下面这些地方尤其明显：

- GrowthBook env override
- prompt template 选择
- internal logging
- 某些看起来像 debug/admin 能力的暴露路径

### 这说明什么

公开包虽然统一发布了，但运行时逻辑里仍然保留了：

- **内部模式**
- **外部模式**

两套行为边界。

因此，那些 hidden/stub 命令很大概率就是内部能力在公开 build 中留下的外壳。

---

## 6. GrowthBook / remote config 又是另一层 gate

feature control 不只停留在 build-time。

恢复出来的代码还显示出这些控制层：

- GrowthBook feature value
- remote-eval cache
- config override
- env var override
- auth-sensitive reinitialize

### 这意味着什么

一个能力可能同时满足下面几层中的若干层：

1. 在源码里存在
2. 被编进某些 build
3. 但运行时依然被 gate/config 关掉

所以真正判断一个能力时，至少要区分：

- 源码里是否出现
- 是否被编进当前 build
- 是否注册成命令
- 是否被 runtime gate 打开
- 是否受 `USER_TYPE` 限制
- 是否还受 env/config 覆盖

这是 reverse engineering 时非常重要的分层。

---

## 7. 不只是“隐藏命令”，有些普通命令其实也分内外版本

`/init` 是一个特别好的例子。

在 `commands/init.ts` 里，它会在两套 prompt 流程之间切换：

- 老版 init prompt
- 新版多阶段 init prompt

切换条件依赖：

- `feature('NEW_INIT')`
- `process.env.USER_TYPE === 'ant'`
- `CLAUDE_CODE_NEW_INIT`

### 这说明什么

即便是一个公开命令，它的**行为本身**也可能有：

- 外部默认版本
- 内部 richer 版本
- 用环境变量强制覆盖的版本

所以这里不只是“某命令是否存在”，而是“同一命令是否有多个 gate 控制的实现版本”。

---

## 8. `TRANSCRIPT_CLASSIFIER`、`KAIROS`、`COORDINATOR_MODE` 很值得注意

这些 feature 名本身就非常有信息量。

### `TRANSCRIPT_CLASSIFIER`
大概率与 classifier 驱动判断有关，尤其和 auto mode / permission automation 一类能力相关。

### `KAIROS`
看起来与 assistant mode 有关。

### `COORDINATOR_MODE`
很像另一种编排/协调者模式，不一定直接暴露在公开 UX 里。

### 这类 feature 名的价值

哪怕公开包里没完全展开具体文档，**feature 名本身就能透露产品方向和内部实验轨迹**。

---

## 9. 目前可以负责任地得出哪些结论

### 高置信度

- 公开包里确实存在 hidden/stubbed command slots
- 一些内部/调试风格的命令名保留在源码树里
- 功能控制至少分成：build-time、user-type、env-var、GrowthBook/runtime 四层
- 外部 build 不一定包含或启用和内部 build 相同的能力

### 中等置信度

- 这些 stub 命令名对应真实存在过或仍存在的内部工作流/工具
- assistant/coordinator/classifier 一类 richer mode 在不同 build 或 gate 下有选择地开放

### 低置信度（需要运行时验证）

- 哪些 hidden command 在任何公开 runtime path 下仍然实际可达
- 是否存在某些非显式入口仍能触发 stub 命令位
- 每个 stub 命令在内部 build 中到底对应什么完整实现

---

## 10. 这对阅读仓库的人意味着什么

如果你是拿这个研究仓库来理解 Claude Code，那么 hidden-command 这一层会改变你的阅读方式：

- 不要默认 `source/src/commands/` 下每个文件都是公开 CLI 可用命令
- 要把 stub 文件和 feature 名当作产品结构线索
- 要重点盯住：
  - `feature(...)`
  - `USER_TYPE === 'ant'`
  - GrowthBook
  - env/config override
- 要明确区分：
  - 源码里存在
  - build 里打进去了
  - runtime 真正启用了

这三层如果混在一起，会很容易误判。

---

## 最后一句话

公开的 `@anthropic-ai/claude-code@2.1.88` 包，通过 hidden stub 命令和多层 feature gate，泄露出了一个更广泛的内部/调试能力面轮廓。最稳妥的理解是：公开 build 保留了内部能力结构的痕迹，但把真正实现裁掉、隐藏或 gate 掉了。
