# learn-claude-code

[中文 README](./README.zh-CN.md)

A research repository for studying `@anthropic-ai/claude-code@2.1.88` from the published npm package.

This repo is built from the released package artifact, its shipped sourcemap, and reconstructed source files. It is intended for reverse engineering, architecture study, and structured documentation — **not** as an official source mirror.

---

## What this repository contains

This repository is organized around four layers:

1. **Raw npm artifacts**
   - the original downloaded package
   - extracted package files such as `cli.js`, `cli.js.map`, and `package.json`

2. **Reconstructed source**
   - source recovered from `cli.js.map`
   - focused primarily on Claude Code’s own source tree

3. **Research documents**
   - bilingual module-by-module analysis documents
   - structure summaries and investigative notes

4. **Repository-level navigation**
   - English + Chinese README
   - reading order and topic index

---

## Why this repo exists

The npm release of `@anthropic-ai/claude-code@2.1.88` shipped with a usable sourcemap (`cli.js.map`) containing embedded `sourcesContent`.

That makes it possible to:

- reconstruct a large portion of the source tree
- inspect architecture decisions from the shipped build
- study how Claude Code handles auth, permissions, multi-agent workflows, MCP, remote sessions, telemetry, and installation/update logic

This repo packages that work into something easier to browse than a raw npm tarball.

---

## Key finding

`@anthropic-ai/claude-code@2.1.88` is **not** just a thin terminal wrapper around a model API.

From the reconstructed source, it appears to include substantial subsystems for:

- OAuth and API-key authentication
- permission approval and risk control
- multi-agent / teammate / swarm orchestration
- deep MCP integration
- remote sessions / bridge / direct-connect execution
- telemetry / analytics / experimentation
- update, installation, and migration workflows

---

## Top findings at a glance

If you only read one section on this page, read this one.

1. **Identity is central to runtime behavior**
   - login/logout affects more than credentials
   - auth transitions refresh policy, feature gates, trusted-device state, and dependent services

2. **Permissions are layered, not binary**
   - modes, allow/deny/ask rules, classifiers, managed policy, and shell-safety checks all interact

3. **Multi-agent is first-class, not experimental garnish**
   - the codebase contains teammate/swarm/task/mailbox/leader-control machinery, not just a single subagent helper

4. **MCP is deeply productized**
   - connection lifecycle, auth, registry, commands, resources, policy filtering, and UI all indicate core-platform status

5. **Remote execution is session-centric and high-trust**
   - bridge workers, session ingress auth, trusted-device tokens, and approval return paths point to a serious remote session plane

6. **Telemetry is operational infrastructure, not just logging**
   - Datadog, first-party event logging, GrowthBook, and OTel-style paths are clearly separated by purpose and privacy boundaries

7. **Distribution is in transition**
   - the code strongly suggests migration from npm-based delivery toward a native installer model

---

## Architecture map

A rough mental model of the recovered system:

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

This is simplified, but it matches the strongest patterns visible in the recovered package.

---

## Start here by reader type

### If you care about security
Start with:
- [Permissions & Risk Control](./doc/permissions-risk-control.en.md)
- [Authentication & Login](./doc/authentication-login.en.md)
- [Remote / Bridge](./doc/remote-bridge.en.md)
- [Telemetry](./doc/telemetry.en.md)

### If you care about product architecture
Start with:
- [Authentication & Login](./doc/authentication-login.en.md)
- [Multi-Agent](./doc/multi-agent.en.md)
- [MCP](./doc/mcp.en.md)
- [Remote / Bridge](./doc/remote-bridge.en.md)

### If you care about reverse engineering / implementation strategy
Start with:
- [Update / Install](./doc/update-install.en.md)
- [MCP](./doc/mcp.en.md)
- [Multi-Agent](./doc/multi-agent.en.md)
- [Telemetry](./doc/telemetry.en.md)

### If you care about enterprise / managed-runtime behavior
Start with:
- [Authentication & Login](./doc/authentication-login.en.md)
- [Permissions & Risk Control](./doc/permissions-risk-control.en.md)
- [MCP](./doc/mcp.en.md)
- [Remote / Bridge](./doc/remote-bridge.en.md)
- [Telemetry](./doc/telemetry.en.md)

---

## Repository layout

### [`npm-original/`](./npm-original/)
Raw package artifacts and extracted release files.

Typical contents include:

- `anthropic-ai-claude-code-2.1.88.tgz`
- `cli.js`
- `cli.js.map`
- `package.json`
- extraction metadata

### [`source/`](./source/)
Reconstructed source tree derived from `cli.js.map`.

This repo focuses on Claude Code’s own source and removes most third-party `node_modules` noise to make browsing easier.

### [`doc/`](./doc/)
Research and analysis documents.

This is the most important directory for readers who want conclusions before code spelunking.

### [`README.md`](./README.md) / [`README.zh-CN.md`](./README.zh-CN.md)
Top-level navigation in English and Chinese.

---

## Documentation entry points

- [Main documentation index (English)](./doc/INDEX.en.md)
- [主文档索引（中文）](./doc/INDEX.zh-CN.md)

If you are new to this repo, I recommend:

1. read this README for repository scope
2. open the [English documentation index](./doc/INDEX.en.md) for structured navigation
3. then choose a reading path by topic

---

## Suggested reading order

If you want one straightforward path, read in this order:

1. [This README](./README.md)
2. [Documentation Index](./doc/INDEX.en.md)
3. [Authentication & Login](./doc/authentication-login.en.md)
4. [Permissions & Risk Control](./doc/permissions-risk-control.en.md)
5. [Multi-Agent](./doc/multi-agent.en.md)
6. [MCP](./doc/mcp.en.md)
7. [Remote / Bridge](./doc/remote-bridge.en.md)
8. [Telemetry](./doc/telemetry.en.md)
9. [Update / Install](./doc/update-install.en.md)

Why this order:

- auth explains identity state
- permissions explains execution control
- multi-agent + MCP explain extensibility and orchestration
- remote/bridge explains high-privilege session execution
- telemetry explains observability and rollout control
- update/install explains delivery and migration strategy

---

## Module analysis index

### 1. Authentication & Login
- [English](./doc/authentication-login.en.md)
- [中文](./doc/authentication-login.zh-CN.md)

Focus:
- OAuth + PKCE
- Claude.ai vs Console login paths
- token source handling
- managed auth context
- trusted-device coupling

### 2. Permissions & Risk Control
- [English](./doc/permissions-risk-control.en.md)
- [中文](./doc/permissions-risk-control.zh-CN.md)

Focus:
- permission modes
- rule engine
- classifier-driven approvals
- managed policy
- Bash security analysis

### 3. Multi-Agent
- [English](./doc/multi-agent.en.md)
- [中文](./doc/multi-agent.zh-CN.md)

Focus:
- AgentTool
- forked subagents
- teammate/swarm runtime
- task graphs
- mailbox and leader approval bridge

### 4. MCP
- [English](./doc/mcp.en.md)
- [中文](./doc/mcp.zh-CN.md)

Focus:
- MCP connection lifecycle
- transports and auth
- official registry
- dynamic tools/commands/resources
- policy-aware extension model

### 5. Remote / Bridge
- [English](./doc/remote-bridge.en.md)
- [中文](./doc/remote-bridge.zh-CN.md)

Focus:
- bridge workers
- session ingress auth
- trusted devices
- direct connect
- remote permission return paths

### 6. Telemetry
- [English](./doc/telemetry.en.md)
- [中文](./doc/telemetry.zh-CN.md)

Focus:
- analytics entry points
- Datadog vs 1P logging
- GrowthBook
- metadata enrichment
- privacy and kill switches

### 7. Update / Install
- [English](./doc/update-install.en.md)
- [中文](./doc/update-install.zh-CN.md)

Focus:
- auto updater
- native installer
- npm global/local migration
- package-manager detection
- version gates and rollout safety

---

## Quick conclusions by topic

### Identity
Claude Code treats auth as a control-plane concern, not just a token lookup.

### Execution safety
Permissions are implemented as a layered system: modes, rules, classifiers, and shell-level safety analysis.

### Orchestration
Multi-agent behavior is first-class, with task graphs, teammate contexts, and leader-mediated control.

### Extensibility
MCP is deeply embedded into commands, tools, permissions, and UI — not a bolt-on plugin interface.

### Remote operation
Remote execution is session-centric and tied to strong identity and device trust.

### Observability
Telemetry is stratified across Datadog, first-party logging, and OTel-style paths, with visible privacy controls.

### Distribution
Installation/update logic reflects a live migration from npm-based delivery toward a native installer model.

---

## Methodology

This repository was built by:

1. downloading the npm package for `@anthropic-ai/claude-code@2.1.88`
2. extracting package contents
3. inspecting `cli.js.map`
4. recovering files from embedded `sourcesContent`
5. separating likely first-party source from bundled third-party code
6. writing structured analysis documents by subsystem

Important caveat:

- reconstructed source comes from the shipped build, not an original development repository
- file names and code content can still be highly informative, but build-time transformations may exist
- some internal-only or feature-gated behavior may be stubbed, absent, or partially represented in the public package

---

## Scope and limitations

This repo is useful for:

- architecture study
- reverse engineering
- security review starting points
- product analysis
- implementation pattern comparison

This repo is **not**:

- an official source release
- guaranteed complete
- guaranteed identical to Anthropic internal builds
- a replacement for runtime testing

---

## Language

- [English README](./README.md)
- [中文 README](./README.zh-CN.md)

---

## Current status

Completed module reports:

- [Authentication & Login](./doc/authentication-login.en.md)
- [Permissions & Risk Control](./doc/permissions-risk-control.en.md)
- [Multi-Agent](./doc/multi-agent.en.md)
- [MCP](./doc/mcp.en.md)
- [Remote / Bridge](./doc/remote-bridge.en.md)
- [Telemetry](./doc/telemetry.en.md)
- [Update / Install](./doc/update-install.en.md)

---

## Package details

- Package: `@anthropic-ai/claude-code`
- Version analyzed: `2.1.88`
- Reconstruction basis: `cli.js.map` with embedded `sources` + `sourcesContent`

---

## Suggested next improvements

Possible future additions to this repo:

- hidden commands / feature flags report
- a dedicated permissions-bypass deep dive
- a remote session protocol note
- a telemetry privacy-boundary note
- an MCP auth deep dive

---

## Disclaimer

This repository is for research, learning, and analysis of a publicly distributed npm package artifact. It is not affiliated with or endorsed by Anthropic.
