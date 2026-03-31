# learn-claude-code

This repository stores a local research copy of `@anthropic-ai/claude-code@2.1.88`, including:

1. **npm-original/**
   - Original npm tarball and key extracted package artifacts
   - `cli.js.map` used for source reconstruction

2. **source/**
   - Reconstructed source tree from `cli.js.map`
   - Focused on Claude Code's own source (third-party `node_modules` removed)

3. **doc/**
   - Analysis reports, module mapping, structure summary, and topic-focused notes

## Language

- English: `README.md`
- 中文：`README.zh-CN.md`

## Layout

- `npm-original/` — raw package artifacts
- `source/` — reconstructed source code
- `doc/` — analysis documents
- `README.md` — English overview
- `README.zh-CN.md` — 中文说明

## Module analysis index

- Authentication & Login
  - English: `doc/authentication-login.en.md`
  - 中文：`doc/authentication-login.zh-CN.md`

- Permissions & Risk Control
  - English: `doc/permissions-risk-control.en.md`
  - 中文：`doc/permissions-risk-control.zh-CN.md`

- Multi-Agent
  - English: `doc/multi-agent.en.md`
  - 中文：`doc/multi-agent.zh-CN.md`

- MCP
  - English: `doc/mcp.en.md`
  - 中文：`doc/mcp.zh-CN.md`

- Remote / Bridge
  - English: `doc/remote-bridge.en.md`
  - 中文：`doc/remote-bridge.zh-CN.md`

- Telemetry
  - English: `doc/telemetry.en.md`
  - 中文：`doc/telemetry.zh-CN.md`

> More module reports will be added incrementally: update/install.

## Notes

- Package: `@anthropic-ai/claude-code`
- Version: `2.1.88`
- Source reconstruction is based on `sources` + `sourcesContent` embedded in `cli.js.map`
- This repo is organized for inspection and learning
- This is a research and analysis repository, not an official source mirror

## Current high-level impression

From the recovered source tree, Claude Code 2.1.88 appears to be much more than a thin terminal wrapper. It includes:

- a full OAuth / API-key authentication stack
- permission approval and risk-control logic
- multi-agent / teammate / swarm collaboration features
- deep MCP integration
- remote-control / bridge / direct-connect capabilities
- telemetry / analytics / experiment infrastructure
- update and local installation logic
