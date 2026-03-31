# 埋点 / Telemetry 模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/services/analytics/index.ts`
- `source/src/services/analytics/sink.ts`
- `source/src/services/analytics/datadog.ts`
- `source/src/services/analytics/firstPartyEventLogger.ts`
- `source/src/services/analytics/growthbook.ts`
- `source/src/services/analytics/metadata.ts`
- `source/src/services/analytics/config.ts`
- `source/src/utils/telemetry/events.ts`

## 总体结论

Claude Code 的埋点系统不是“到处调用一个 logEvent()”那么简单，而是一套分层的 telemetry / analytics 基础设施，至少包括：

1. **统一埋点入口**
2. **sink 路由层**
3. **Datadog 通道**
4. **1P（自有）事件日志通道**
5. **GrowthBook / feature gate / dynamic config / experiment exposure**
6. **metadata enrichment 与敏感信息约束**
7. **OTel 事件路径**
8. **sampling、killswitch、privacy level、provider gating**

也就是说，这一块不是附属日志模块，而是一个已经相当成熟的数据采集与实验控制底座。

---

## 1. `analytics/index.ts`：埋点公共入口非常克制

这个文件的设计很有意思：

- 公开 API 只有 `logEvent()` / `logEventAsync()` / `attachAnalyticsSink()`
- 模块刻意保持**无依赖**，避免 import cycle
- 在 sink 初始化之前，事件会先进入队列
- sink attach 后再异步 drain queue

### 这说明什么

作者明显想把埋点入口做成：

- **超稳定**
- **可早期调用**
- **不把复杂 sink 依赖传播到全项目**

### 一个重要设计点

metadata 类型非常严格：

- 默认只接受 `boolean | number | undefined`
- 字符串需要显式标记“我确认这不是代码/路径/敏感内容”
- `_PROTO_*` 字段还会被专门剥离，避免误流入普通后端

### 结论

Claude Code 在埋点入口层就明显有“防止敏感数据误上报”的设计意识。

---

## 2. `sink.ts`：统一路由 + 采样 + killswitch

`sink.ts` 是真正的路由层。

### 它做的事情

- 根据采样规则决定事件要不要上报
- 根据 gate 决定 Datadog 是否启用
- 对 Datadog 路径剥离 `_PROTO_*` 字段
- 把完整 payload 留给 1P sink
- 提供 `initializeAnalyticsGates()` 和 `initializeAnalyticsSink()`

### 这里最关键的点

#### 1）Datadog 和 1P 不是同权后端

- Datadog 是 general-access backend
- 1P 是 privileged backend

这就是为什么 `_PROTO_*` 字段：

- 在 Datadog 前会 strip
- 在 1P 路径里还能保留/提升到 proto field

#### 2）有 killswitch

`sink.ts` 会检查：

- `isSinkKilled('datadog')`
- 以及对应 1P sink 的 kill 逻辑

说明这不是死板的单向上报，而是可以远程熔断的。

#### 3）采样是全局内建的

不是每个 call site 自己 decide，而是 sink 统一走 `shouldSampleEvent()`。

### 结论

`sink.ts` 的存在说明 Claude Code 不是“散装埋点”，而是明确区分了：

- 入口
- enrich
- 路由
- sink
- 采样
- 熔断

这几个层次。

---

## 3. Datadog：不是全量镜像，而是严格白名单 + 降基数

`analytics/datadog.ts` 很值得看，因为它透露出很强的生产环境经验。

### 明显特征

- Datadog endpoint 和 client token 写死在代码里
- 有 `DATADOG_ALLOWED_EVENTS` 白名单
- 只有 production 才发
- 只对 first-party provider 发
- 有 batch + flush interval + max batch size + network timeout

### 关键观察 1：它不是“所有事件都发 Datadog”

白名单中只挑了一部分事件，例如：

- OAuth 成功/失败
- tool use 成功/失败
- API 成功/失败
- voice toggle
- team mem sync
- crash / exception

这说明 Datadog 更多承担：

- 运维观察
- 故障监控
- 核心用户行为指标

而不是作为全量事件仓库。

### 关键观察 2：非常强调 cardinality 控制

它专门做了这些事：

- MCP tool 名统一收敛成 `mcp`
- 外部用户模型名归一成 canonical short name / `other`
- dev version 去掉 timestamp 和 sha
- `status` 转成 `http_status` 和 `http_status_range`
- 通过 `TAG_FIELDS` 控制可搜索字段

这说明作者非常清楚 Datadog 最怕什么：

- tag cardinality 爆炸
- 版本号爆炸
- tool 名 / model 名无限扩散

### 关键观察 3：支持 shutdown flush

`shutdownDatadog()` 会在退出前 flush remaining logs。

这也解释了为什么 bridge/logout/shutdown 等地方会专门调用它。

### 结论

Datadog 在 Claude Code 里更像：

- 低延迟运维观测通道
- 经过白名单与降基数严格约束

而不是“通用事件总线”。

---

## 4. 1P Event Logging：这才是更完整的内部事件通道

`firstPartyEventLogger.ts` 说明 Claude Code 还有一条更重的内部事件链路。

### 它的特点

- 基于 OTel logs SDK
- `LoggerProvider + BatchLogRecordProcessor`
- 自定义 exporter：`FirstPartyEventLoggingExporter`
- 同样支持 batch config，但配置来自 GrowthBook dynamic config
- 会把 core metadata / user metadata / event metadata 作为结构化属性送出去

### 这里最关键的点

#### 1）支持 sampling config

- `tengu_event_sampling_config`

这说明事件采样不只是 Datadog 上的考量，而是更上层的统一策略。

#### 2）batch config 可动态下发

- `tengu_1p_event_batch_config`

包括：

- `scheduledDelayMillis`
- `maxExportBatchSize`
- `maxQueueSize`
- `skipAuth`
- `maxAttempts`
- `path`
- `baseUrl`

这说明 1P event logger 的行为能被远程配置，而不是写死。

#### 3）GrowthBook exposure 也走 1P

里面专门有：

- `logGrowthBookExperimentTo1P()`

说明实验分流和事件日志是闭环的，不是两套脱节系统。

### 结论

如果说 Datadog 是运维和聚合搜索通道，那 1P event logging 更像：

- 内部精细化分析通道
- 实验/业务分析数据源

---

## 5. `growthbook.ts`：feature flag、动态配置、实验暴露，是 telemetry 的另一半

这个文件非常大，也非常关键。

### 它做的不只是 feature gate

从代码看，它至少管理：

- GrowthBook client lifecycle
- auth change 后重建 client
- remote eval feature value cache
- env override / config override
- exposure logging
- refresh listener
- reinitializing promise
- disk cache / stale cache fallback
- user attributes for targeting

### 关键观察 1：GrowthBook 不只是“if feature on/off”

它还承担：

- dynamic config 分发
- 采样配置
- batch 配置
- 安全 gate
- experiment exposure 记录

也就是说，GrowthBook 在 Claude Code 里既是：

- 功能开关系统
- 也是运行时控制平面的一部分

### 关键观察 2：非常在意 stale vs blocking

你会反复看到：

- `CACHED_MAY_BE_STALE`
- `CACHED_OR_BLOCKING`

这说明作者明确在平衡：

- 启动速度
- gate 正确性
- auth change 后的一致性

特别是安全相关的 gate，会尽量走 blocking path，不依赖陈旧缓存。

### 关键观察 3：覆盖体系很完整

支持：

- env var override（ant-only）
- config override
- remote eval
- disk cache fallback

这对：

- 内部测试
- 评测 harness
- 本地 debug
- rollout 管控

都很有用。

### 结论

GrowthBook 在这里不只是 A/B test SDK，而是 Claude Code 整个运行时的动态配置中枢之一。

---

## 6. `metadata.ts`：埋点不是“原样上报”，而是高度加工后的结构化元数据

这个文件展示了 Claude Code 对 telemetry 的另一种成熟度：

- 很多元数据都不是 call site 直接传，而是统一 enrich

### 它会处理的东西包括

- session / parentSession / team / teammate 相关上下文
- model / betas / provider / platform / WSL / distro
- interactive / clientType / kairos active
- git / repo remote hash / VCS
- auth/subscription 信息
- MCP tool 细节是否允许上报
- skill 名提取
- tool input 截断 / 限深 / 限集合大小

### 一个非常关键的点

MCP 细节不是默认全上报。

只有满足一定条件时才会上报具体 server/tool 名，例如：

- local-agent/cowork 场景
- claude.ai proxy
- official registry 中的 MCP

否则只会保留更泛化的字段。

### 这说明什么

作者在 telemetry 里明显试图平衡：

- 诊断价值
- PII 风险
- 用户自定义 MCP 配置暴露风险

### 结论

`metadata.ts` 说明 Claude Code 的 telemetry 已经不是“字段越多越好”，而是做过隐私分级和可观测性取舍。

---

## 7. `utils/telemetry/events.ts`：还有一条 OTel 风格 3P telemetry 路线

这个文件走的是事件 logger 路径：

- `logOTelEvent()`

### 其特征

- 使用 `eventLogger.emit()`
- 自动加：
  - `event.name`
  - `event.timestamp`
  - `event.sequence`
  - `prompt.id`
  - workspace host paths（事件级，不进 metrics）
- `OTEL_LOG_USER_PROMPTS` 控制 prompt 内容是否 redact

### 这说明什么

Claude Code 至少存在三层事件路径：

1. Datadog
2. 1P event logging
3. OTel / 3P telemetry event logger

这不是重复建设，而是不同目的：

- 运维检索
- 内部分析
- 标准化遥测 / enterprise export

---

## 8. `config.ts`：provider / privacy level / telemetry opt-out 是全局前置条件

`analytics/config.ts` 很短，但非常关键。

### analytics 会在这些场景下整体关闭

- test environment
- Bedrock
- Vertex
- Foundry
- `isTelemetryDisabled()`

### 这意味着什么

Claude Code 并不是“默认所有提供商都一样上报”。

对于：

- 第三方 provider
- 高隐私模式

会直接从总开关层面禁用 analytics。

这也是一个很明确的产品边界：

- telemetry 是 first-party experience 的一部分
- 不是无条件跨 provider 复制

---

## 9. telemetry 在 Claude Code 中的真实角色

综合这些文件，我认为 telemetry 在 Claude Code 中承担了至少四层角色：

### A. 运行观测

- API 成功/失败
- tool use 成功/失败
- crash / reconnect / bridge heartbeat 等

### B. 产品实验与 rollout

- GrowthBook gates
- dynamic config
- experiment exposure
- feature refresh

### C. 内部行为分析

- 1P structured event logging
- richer metadata
- user/session/environment context

### D. 企业/标准化输出

- OTel event logger
- prompt redaction
- workspace path handling

这说明 telemetry 不是附加统计，而是 Claude Code 运行时控制与观察系统的一部分。

---

## 10. 模块设计判断

### 我认为这个模块最重要的特点有四个

#### 1）强约束而不是放任上报

入口类型限制、PII 标记、_PROTO_ 剥离、tool/MCP 细节门控，都说明这套系统不是“先全打了再说”。

#### 2）多后端明确分层

- Datadog：偏运维、白名单、低基数
- 1P：偏内部分析、结构化、可配置批处理
- OTel：偏标准化事件出口

#### 3）实验系统和埋点系统是深度耦合的

GrowthBook 不只是 feature flag，而是采样、batch config、实验曝光、运行时控制的一部分。

#### 4）对真实生产问题很敏感

从 Datadog cardinality 控制、OAuth/gate stale 问题、shutdown flush、provider/privacy opt-out 都能看出来，作者是在处理真实线上系统，而不是 demo 级埋点。

---

## 11. 值得继续深挖的问题

后续如果继续精读，我建议优先追这些：

1. `firstPartyEventLoggingExporter.ts` 的最终上传格式与脱敏策略
2. `sinkKillswitch.ts` 的远程熔断策略
3. `getEventMetadata()` 各字段的精确来源与缓存策略
4. GrowthBook exposure 在哪些 hot path 上会被触发
5. OTel event logger 的最终 exporter 去向
6. `_PROTO_*` 字段有哪些具体映射目标
7. provider/privacy level 对不同 telemetry 路径的差异化影响

---

## 一句话总结

**Claude Code 的 telemetry 模块本质上是一套受约束的数据与实验基础设施：它把事件上报、隐私控制、动态配置、实验分流和多后端输出组织成了一个成熟的运行时观测层。**
