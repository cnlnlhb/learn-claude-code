# Documentation Index

[Back to README](../README.md) · [中文索引](./INDEX.zh-CN.md)

This page is the structured navigation hub for the research documents in this repository.

---

## Start here

If you are new to the repo, start in this order:

1. [README](../README.md)
2. [Authentication & Login](./authentication-login.en.md)
3. [Permissions & Risk Control](./permissions-risk-control.en.md)
4. [Multi-Agent](./multi-agent.en.md)
5. [MCP](./mcp.en.md)
6. [Remote / Bridge](./remote-bridge.en.md)
7. [Telemetry](./telemetry.en.md)
8. [Update / Install](./update-install.en.md)

---

## Reading paths

### Path A — Claude Code as a product runtime

1. [Authentication & Login](./authentication-login.en.md)
2. [Permissions & Risk Control](./permissions-risk-control.en.md)
3. [Remote / Bridge](./remote-bridge.en.md)
4. [Telemetry](./telemetry.en.md)
5. [Update / Install](./update-install.en.md)

Why:
- identity explains who the runtime thinks you are
- permissions explains what the runtime lets you do
- remote/bridge explains how execution leaves the local machine
- telemetry explains how behavior is observed and controlled
- update/install explains how the client is delivered and migrated

### Path B — Claude Code as an orchestration system

1. [Multi-Agent](./multi-agent.en.md)
2. [MCP](./mcp.en.md)
3. [Remote / Bridge](./remote-bridge.en.md)
4. [Permissions & Risk Control](./permissions-risk-control.en.md)

Why:
- multi-agent explains worker orchestration
- MCP explains extension and capability injection
- remote/bridge explains session transport and control
- permissions explains how execution authority is constrained

### Path C — Claude Code as a secure execution client

1. [Authentication & Login](./authentication-login.en.md)
2. [Permissions & Risk Control](./permissions-risk-control.en.md)
3. [Remote / Bridge](./remote-bridge.en.md)
4. [Telemetry](./telemetry.en.md)

Why:
- auth controls identity
- permissions controls local execution
- remote/bridge controls remote execution
- telemetry reveals enforcement, rollout, and incident-response patterns

---

## Module portal

### Authentication & Login
- [English](./authentication-login.en.md)
- [中文](./authentication-login.zh-CN.md)

Topics:
- OAuth + PKCE
- Claude.ai vs Console paths
- managed auth context
- token source hierarchy
- trusted-device coupling

### Permissions & Risk Control
- [English](./permissions-risk-control.en.md)
- [中文](./permissions-risk-control.zh-CN.md)

Topics:
- permission modes
- allow/deny/ask rule engine
- classifier-based approval
- managed policy override
- Bash/zsh risk analysis

### Multi-Agent
- [English](./multi-agent.en.md)
- [中文](./multi-agent.zh-CN.md)

Topics:
- AgentTool
- fork subagents
- teammate/swarm runtime
- task lists
- mailbox and leader approval bridge

### MCP
- [English](./mcp.en.md)
- [中文](./mcp.zh-CN.md)

Topics:
- MCP connection lifecycle
- auth and transport
- official registry
- tools / commands / resources
- policy-aware extension model

### Remote / Bridge
- [English](./remote-bridge.en.md)
- [中文](./remote-bridge.zh-CN.md)

Topics:
- bridge workers
- session ingress auth
- trusted device
- direct connect
- remote approval/control loops

### Telemetry
- [English](./telemetry.en.md)
- [中文](./telemetry.zh-CN.md)

Topics:
- analytics entry layer
- Datadog vs 1P event logging
- GrowthBook
- metadata enrichment
- privacy controls and kill switches

### Update / Install
- [English](./update-install.en.md)
- [中文](./update-install.zh-CN.md)

Topics:
- auto updater
- native installer
- npm global/local migration
- package-manager detection
- rollout/version gates

---

## Supporting materials

- [README (English)](../README.md)
- [README (中文)](../README.zh-CN.md)
- [npm-original/](../npm-original/)
- [source/](../source/)

---

## Suggested future additions

Useful future companion notes for this repo would be:

- hidden commands / feature flags
- permission-bypass paths
- remote session protocol details
- telemetry privacy boundaries
- MCP auth deep dive
