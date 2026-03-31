# Documentation Index

This index provides a structured entry point into the research documents in this repository.

If you are new here, start with:

1. `../README.md`
2. `authentication-login.en.md`
3. `permissions-risk-control.en.md`
4. `multi-agent.en.md`
5. `mcp.en.md`
6. `remote-bridge.en.md`
7. `telemetry.en.md`
8. `update-install.en.md`

---

## Recommended reading paths

### Path A — Understand Claude Code as a product runtime

1. Authentication & Login
2. Permissions & Risk Control
3. Remote / Bridge
4. Telemetry
5. Update / Install

Why:
- identity explains who the runtime thinks you are
- permissions explains what the runtime lets you do
- remote/bridge explains how execution leaves the local machine
- telemetry explains how behavior is observed and controlled
- update/install explains how the client is delivered and migrated

### Path B — Understand Claude Code as an orchestration system

1. Multi-Agent
2. MCP
3. Remote / Bridge
4. Permissions & Risk Control

Why:
- multi-agent explains worker orchestration
- MCP explains extension and capability injection
- remote/bridge explains session transport and control
- permissions explains how execution authority is constrained

### Path C — Understand Claude Code as a secure execution client

1. Authentication & Login
2. Permissions & Risk Control
3. Remote / Bridge
4. Telemetry

Why:
- auth controls identity
- permissions controls local execution
- remote/bridge controls remote execution
- telemetry reveals enforcement, rollout, and incident-response patterns

---

## Module documents

### Authentication & Login
- `authentication-login.en.md`
- `authentication-login.zh-CN.md`

Topics:
- OAuth + PKCE
- Claude.ai vs Console paths
- managed auth context
- token source hierarchy
- trusted-device coupling

### Permissions & Risk Control
- `permissions-risk-control.en.md`
- `permissions-risk-control.zh-CN.md`

Topics:
- permission modes
- allow/deny/ask rule engine
- classifier-based approval
- managed policy override
- Bash/zsh risk analysis

### Multi-Agent
- `multi-agent.en.md`
- `multi-agent.zh-CN.md`

Topics:
- AgentTool
- fork subagents
- teammate/swarm runtime
- task lists
- mailbox and leader approval bridge

### MCP
- `mcp.en.md`
- `mcp.zh-CN.md`

Topics:
- MCP connection lifecycle
- auth and transport
- official registry
- tools / commands / resources
- policy-aware extension model

### Remote / Bridge
- `remote-bridge.en.md`
- `remote-bridge.zh-CN.md`

Topics:
- bridge workers
- session ingress auth
- trusted device
- direct connect
- remote approval/control loops

### Telemetry
- `telemetry.en.md`
- `telemetry.zh-CN.md`

Topics:
- analytics entry layer
- Datadog vs 1P event logging
- GrowthBook
- metadata enrichment
- privacy controls and kill switches

### Update / Install
- `update-install.en.md`
- `update-install.zh-CN.md`

Topics:
- auto updater
- native installer
- npm global/local migration
- package-manager detection
- rollout/version gates

---

## Supporting materials already in repo

These are useful if you want structure before details:

- `../README.md`
- `../README.zh-CN.md`
- `../npm-original/`
- `../source/`

---

## Suggested future additions

Useful future companion notes for this repo would be:

- hidden commands / feature flags
- permission-bypass paths
- remote session protocol details
- telemetry privacy boundaries
- MCP auth deep dive
