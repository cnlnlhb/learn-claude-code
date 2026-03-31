# learn-claude-code

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

## Repository layout

### `npm-original/`
Raw package artifacts and extracted release files.

Typical contents include:

- `anthropic-ai-claude-code-2.1.88.tgz`
- `cli.js`
- `cli.js.map`
- `package.json`
- extraction metadata

### `source/`
Reconstructed source tree derived from `cli.js.map`.

This repo focuses on Claude Code’s own source and removes most third-party `node_modules` noise to make browsing easier.

### `doc/`
Research and analysis documents.

This is the most important directory for readers who want conclusions before code spelunking.

### `README.md` / `README.zh-CN.md`
Top-level navigation in English and Chinese.

---

## Documentation entry points

- Main documentation index (English): `doc/INDEX.en.md`
- 主文档索引（中文）: `doc/INDEX.zh-CN.md`

If you are new to this repo, I recommend:

1. read this README for repository scope
2. open `doc/INDEX.en.md` for structured navigation
3. then choose a reading path by topic

---

## Suggested reading order

If you want one straightforward path, read in this order:

1. **This README**
2. **Documentation Index**
3. **Authentication & Login**
4. **Permissions & Risk Control**
5. **Multi-Agent**
6. **MCP**
7. **Remote / Bridge**
8. **Telemetry**
9. **Update / Install**

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
- English: `doc/authentication-login.en.md`
- 中文：`doc/authentication-login.zh-CN.md`

Focus:
- OAuth + PKCE
- Claude.ai vs Console login paths
- token source handling
- managed auth context
- trusted-device coupling

### 2. Permissions & Risk Control
- English: `doc/permissions-risk-control.en.md`
- 中文：`doc/permissions-risk-control.zh-CN.md`

Focus:
- permission modes
- rule engine
- classifier-driven approvals
- managed policy
- Bash security analysis

### 3. Multi-Agent
- English: `doc/multi-agent.en.md`
- 中文：`doc/multi-agent.zh-CN.md`

Focus:
- AgentTool
- forked subagents
- teammate/swarm runtime
- task graphs
- mailbox and leader approval bridge

### 4. MCP
- English: `doc/mcp.en.md`
- 中文：`doc/mcp.zh-CN.md`

Focus:
- MCP connection lifecycle
- transports and auth
- official registry
- dynamic tools/commands/resources
- policy-aware extension model

### 5. Remote / Bridge
- English: `doc/remote-bridge.en.md`
- 中文：`doc/remote-bridge.zh-CN.md`

Focus:
- bridge workers
- session ingress auth
- trusted devices
- direct connect
- remote permission return paths

### 6. Telemetry
- English: `doc/telemetry.en.md`
- 中文：`doc/telemetry.zh-CN.md`

Focus:
- analytics entry points
- Datadog vs 1P logging
- GrowthBook
- metadata enrichment
- privacy and kill switches

### 7. Update / Install
- English: `doc/update-install.en.md`
- 中文：`doc/update-install.zh-CN.md`

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

- English: `README.md`
- 中文：`README.zh-CN.md`

---

## Current status

Completed module reports:

- Authentication & Login
- Permissions & Risk Control
- Multi-Agent
- MCP
- Remote / Bridge
- Telemetry
- Update / Install

---

## Package details

- Package: `@anthropic-ai/claude-code`
- Version analyzed: `2.1.88`
- Reconstruction basis: `cli.js.map` with embedded `sources` + `sourcesContent`

---

## Suggested next improvements

Possible future additions to this repo:

- a hidden commands / feature flags report
- a dedicated permissions-bypass deep dive
- a remote session protocol note
- a telemetry privacy-boundary note
- an MCP auth deep dive

---

## Disclaimer

This repository is for research, learning, and analysis of a publicly distributed npm package artifact. It is not affiliated with or endorsed by Anthropic.
