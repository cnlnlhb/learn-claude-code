# 更新 / 安装模块分析（中文）

## 模块范围

本模块主要对应以下源码入口：

- `source/src/components/AutoUpdater.tsx`
- `source/src/utils/autoUpdater.ts`
- `source/src/utils/localInstaller.ts`
- `source/src/utils/nativeInstaller/index.ts`
- `source/src/utils/nativeInstaller/installer.ts`
- `source/src/utils/nativeInstaller/download.ts`
- `source/src/utils/nativeInstaller/packageManagers.ts`
- `source/src/commands/install.tsx`
- `source/src/commands/upgrade/upgrade.tsx`

## 总体结论

Claude Code 的更新 / 安装体系不是简单的 `npm i -g` 包装，而是一套正在从 JS/npm 安装模型迁移到 **native installer** 的多路径分发系统，至少包括：

1. **自动更新 UI 与后台定时检查**
2. **JS/npm global 更新路径**
3. **JS/npm local 更新路径**
4. **native installer 路径**
5. **按平台下载二进制 / 原生包**
6. **多进程锁、版本保留、原子切换、shell 集成**
7. **安装来源识别（brew / winget / deb / rpm / pacman / asdf / mise / npm）**
8. **最小版本 gate / 最大版本 kill switch / channel 管理**

这说明“安装与升级”在 Claude Code 里已经是一个独立的运行时子系统，而不是附属脚本。

---

## 1. `AutoUpdater.tsx`：更新逻辑不是隐藏后台脚本，而是 UI 驱动状态机

`AutoUpdater.tsx` 很直白地展示了自动更新的产品形态。

### 它做的事情

- 初次进入时立即检查更新
- 之后每 30 分钟再检查一次
- 读取当前版本和最新版本
- 查询 server-side `maxVersion`
- 根据当前安装类型选择更新方法
- 把更新结果映射成 UI 状态和 telemetry 事件

### 一个关键点

它不是写死只走一种更新方式，而是先调用：

- `getCurrentInstallationType()`

然后区分：

- `npm-local`
- `npm-global`
- `native`
- `development`
- unknown fallback

### 这说明什么

Claude Code 明显处于一个**多安装来源并存**的阶段，所以 auto updater 必须先识别“我现在到底是怎么装上的”，再决定如何更新。

### 另一个关键点

如果当前不是 native 安装，它会先：

- `removeInstalledSymlink()`

说明 native installer 和旧 JS updater 在 launcher/symlink 层面是会互相干扰的，因此更新流程里专门做了迁移清理。

---

## 2. `autoUpdater.ts`：JS 更新链路和版本 gate 都在这里

这个文件是 JS 安装路径下的核心更新逻辑。

### 它负责的事情包括

- `assertMinVersion()`：最小版本硬门槛
- `getMaxVersion()`：最大版本 kill switch
- `getMaxVersionMessage()`：事故提示文案
- `shouldSkipVersion()`：基于 minimumVersion 防降级
- update lock 文件
- 拉取最新版本
- npm install / global install

### 关键观察 1：minVersion 是强制挡板

如果当前版本低于 `tengu_version_config.minVersion`，CLI 会直接提示必须运行：

- `claude update`

这说明他们已经把“版本过旧无法继续工作”做成了服务端可控策略，而不是仅靠公告提醒。

### 关键观察 2：maxVersion 是 server-side kill switch

如果最新版本高于允许的 `maxVersion`，auto updater 会把升级目标 cap 到这个版本，甚至完全跳过升级。

### 这意味着什么

安装/升级系统不仅负责“往前走”，还负责：

- **在事故时停住**
- **防止坏版本继续扩散**

这在成熟客户端分发系统里很常见。

### 关键观察 3：更新锁做得很认真

这里专门有：

- `.update.lock`
- stale lock 接管逻辑
- TOCTOU 竞争说明
- ENOENT / EEXIST 竞争处理

说明他们已经遇到或预期会遇到：

- 多个 claude 进程同时尝试升级
- 旧锁遗留
- mkdir/write lock race

### 结论

`autoUpdater.ts` 说明更新系统已经具备：

- 版本控制面
- 故障熔断
- 并发安全

这三个成熟分发系统才会明显重视的特征。

---

## 3. local installer：为了避开全局 npm 权限问题的过渡路径

`localInstaller.ts` 很清楚地体现了一个现实：

- 用户环境里的 `npm -g` 并不总是可靠

### 它的做法

- 在 `~/.claude/local` 下建立私有安装目录
- 自动写 `package.json`
- 自动写一个 wrapper 脚本 `claude`
- 用 `npm install @...` 更新到本地目录
- 成功后把 `installMethod` 标记为 `local`

### 设计意义

这其实是一个折中方案：

- 还保留 npm 包分发
- 但绕开系统全局目录权限、sudo、环境污染等问题

### 结论

local installer 可以看作 native installer 之前/并行阶段的“过渡稳定方案”。

---

## 4. native installer：这是更像正式客户端分发系统的一条路

`nativeInstaller/installer.ts` 是整个更新/安装模块里最重的文件之一。

### 注释已经说明了它想做什么

- 目录结构管理
- symlink activation
- 多进程安全
- fallback mechanism
- 同时支持 JS 和 native build

### 目录结构很清楚

- data: versions
- cache: staging
- state: locks
- user bin: executable

也就是：

- 持久版本目录
- 临时下载/解包目录
- 锁目录
- 用户可执行入口

### 它体现出的核心能力

- 按平台生成 binary name
- 下载后 staged -> install path 原子移动
- retain old versions
- cleanup stale locks
- process lifetime lock
- mtime lock fallback
- setup PATH / alias / shell integration
- 清理旧 npm 安装与 alias

### 一个非常关键的点

它不是只管“把文件放进去”，而是整个 lifecycle 都覆盖：

- install
- activate
- hold lock
- cleanup
- retain previous versions
- shell integration
- migrate from old methods

### 结论

native installer 是 Claude Code 真正“客户端化”的标志之一。

---

## 5. `download.ts`：下载来源是分用户类型分流的

这个文件说明下载路径并不是单一 URL。

### 当前可见两类来源

#### ant 用户

- Artifactory
- platform-specific native package
- `npm view ... dist.integrity`
- `npm ci` + integrity 验证

#### external 用户

- GCS bucket
- `claude-code-releases`

### 关键观察 1：支持直接 version / stable / latest

`getLatestVersion()` 同时支持：

- 明确版本号
- `stable`
- `latest`

而且对特殊测试版本有 feature gate 限制。

### 关键观察 2：对完整性校验很重视

Artifactory 路径会：

- 获取平台包的 `dist.integrity`
- 写 package-lock
- 再让 npm install/ci 做 integrity 校验

这说明他们不是简单 curl 一个包就执行，而是尽量借助 npm 生态已有的完整性机制。

### 关键观察 3：下载也打 telemetry

例如：

- `tengu_version_check_success`
- `tengu_version_check_failure`

这和前面 telemetry 文档是呼应的：更新链路本身就是重点可观测对象。

---

## 6. `packageManagers.ts`：安装来源识别是迁移策略的重要基础

这个文件很容易被忽略，但其实很关键。

### 它检测的来源包括

- Homebrew
- winget
- pacman
- deb
- rpm
- apk
- mise
- asdf
- unknown

### 为什么重要

因为 Claude Code 明显在做“多来源统一迁移”。

如果不知道当前实例是：

- brew 装的
- winget 装的
- npm 装的
- mise/asdf 管的

就没法安全地决定：

- 应不应该自更新
- 应不应该提示改用系统包管理器
- 应不应该清理旧安装

### 结论

安装来源检测不是装饰功能，而是分发治理和迁移兼容的前置能力。

---

## 7. `/install` 命令：native install 是产品化入口

`commands/install.tsx` 把 native install 暴露成了显式命令入口。

### 它的流程

1. 确定 target（latest / stable / specific version）
2. 调 `installLatest(channelOrVersion, force)`
3. 完成后跑 `checkInstall(true)` 做 launcher 与 shell integration
4. 然后执行：
   - `cleanupNpmInstallations()`
   - `cleanupShellAliases()`
5. 成功后更新 settings（如 autoUpdatesChannel）
6. 通过状态机给用户展示安装、setup、success、error

### 这说明什么

`/install` 不只是下载程序，而是一个**迁移命令**：

- 安装 native build
- 修 shell 启动入口
- 清理旧 npm 安装
- 清理旧 alias
- 保存后续更新 channel

也就是说，它实际上承担了“从旧安装模型迁到新安装模型”的产品责任。

---

## 8. `/upgrade` 命令：并不是更新二进制，而是升级 Claude.ai 订阅计划

`commands/upgrade/upgrade.tsx` 很有意思，因为它和“程序升级”不是一回事。

### 它做的事情

- 检查当前是否已经是最高 Max 计划
- 如果不是，打开：
  - `https://claude.ai/upgrade/max`
- 然后直接进入一轮新的登录流程

### 这说明什么

在 Claude Code 里，“upgrade”这个词同时有两层意思：

1. **软件版本升级**（install/update 路径）
2. **账号订阅升级**（Claude.ai Max 路径）

这也是产品层一个挺有意思的设计点：

- 同一个 CLI 同时管理客户端分发和账号能力升级

---

## 9. 更新 / 安装系统在 Claude Code 中的真实角色

综合这些文件，我认为它承担了至少四层角色：

### A. 分发系统

- 版本查询
- channel 选择
- 安装包下载
- binary staging / activation

### B. 迁移系统

- npm global -> local/native
- shell alias 清理
- launcher 修复
- symlink 管理

### C. 兼容系统

- 安装来源识别
- 多 provider of install method
- 不同平台二进制名与路径

### D. 风险控制系统

- minVersion 强制升级
- maxVersion kill switch
- stale lock / concurrent update safety
- development build 禁止 auto-update

这说明更新 / 安装并不是“附赠脚本”，而是一个完整的客户端交付层。

---

## 10. 模块设计判断

### 我认为这个模块最重要的特点有四个

#### 1）明显处在“从 npm CLI 向 native 客户端分发迁移”的阶段

代码里同时存在：

- npm global
- npm local
- native installer
- 旧 alias / symlink cleanup

这几条路径并存，非常像迁移期产品。

#### 2）安装更新逻辑已经高度产品化

UI 状态、路径检测、settings 存储、shell integration、cleanup、telemetry 都在里面，不是脚本堆砌。

#### 3）并发与故障控制做得相对成熟

多个锁实现、stale lock 接管、atomic move、版本保留、max/min version gate，都说明这是被当成线上分发系统来设计的。

#### 4）安装系统和账号系统都被纳入 CLI 生命周期

不仅客户端版本会升级，连 Claude.ai 订阅升级和重新登录流程也被纳入 CLI 命令体系。

---

## 11. 值得继续深挖的问题

后续如果继续精读，我建议优先追这些：

1. `installLatest()` 的完整激活与回滚路径
2. 原生二进制下载后的签名/校验边界
3. `checkInstall()` 如何具体修改 PATH、alias、shell 配置
4. `cleanupNpmInstallations()` 对不同系统包管理器的处理边界
5. native installer 与 direct package manager（brew/winget）之间未来是否会冲突
6. GCS 与 Artifactory 两条分发链路的版本一致性策略
7. 版本保留与清理策略在磁盘不足场景下的行为

---

## 一句话总结

**Claude Code 的更新 / 安装模块本质上是一套正在客户端化的分发与迁移系统：它把版本控制、下载、安装、激活、并发保护、环境修复和旧安装清理整合进了同一个产品化交付层。**
