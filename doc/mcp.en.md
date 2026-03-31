# MCP Module Analysis

## Scope

This analysis primarily covers:

- `source/src/services/mcp/MCPConnectionManager.tsx`
- `source/src/services/mcp/useManageMCPConnections.ts`
- `source/src/commands/mcp/mcp.tsx`
- `source/src/services/mcp/auth.ts`
- `source/src/services/mcp/mcpStringUtils.ts`
- `source/src/services/mcp/officialRegistry.ts`
- `source/src/utils/mcpWebSocketTransport.ts`
- `source/src/components/mcp/MCPSettings.tsx`
- `source/src/components/mcp/MCPReconnect.tsx`

## Executive Summary

Claude Code’s MCP support is much more than “connect to an external tool server.” It is a deeply integrated extension layer that includes at least:

1. **connection lifecycle management**
2. **multiple transports** (stdio / SSE / HTTP / WebSocket / Claude.ai proxy)
3. **OAuth / OIDC / token refresh / secure storage**
4. **unified loading of MCP tools, commands, and resources**
5. **configuration, enable/disable, reconnect, and auth-state UI**
6. **policy filtering, official registry lookups, and channel / permission relay support**
7. **tight integration with agents, permissions, plugins, and remote sessions**

So in Claude Code, MCP is not a peripheral plugin API. It is one of the core extension buses.

---

## 1. `MCPConnectionManager`: connection control plane

`MCPConnectionManager.tsx` is simple, but it defines the core control-plane abstraction:

- `reconnectMcpServer(serverName)`
- `toggleMcpServer(serverName)`

These are then exposed via React context to UI and command surfaces.

### Interpretation

MCP is not something the runtime merely initializes once. It is designed to be controllable at runtime:

- reconnectable
- enable/disable-able
- operable from settings views and `/mcp` commands

---

## 2. The real runtime lives in `useManageMCPConnections.ts`

This hook is the real MCP runtime manager.

### Responsibilities visible in the recovered code

- initialize MCP connections
- synchronize them into AppState
- perform automatic reconnection
- dynamically refresh tools / commands / resources
- refresh plugin-backed MCP servers
- fetch Claude.ai MCP configuration if eligible
- apply enterprise policy / config filtering
- deduplicate errors and notifications
- handle channel permission relay
- register elicitation handlers
- batch AppState updates for connection churn

### Particularly important observations

#### 1) MCP contributes more than tools

The runtime tracks:

- `mcp.clients`
- `mcp.tools`
- `mcp.commands`
- `mcp.resources`

So MCP is not merely “external tools.” It is a full protocol surface inside Claude Code.

#### 2) Connections are dynamic, not static

The code includes:

- reconnect backoff
- connection lifecycle handling
- list-changed notification handling
- incremental refresh of tool/resource/command state

This means MCP servers are treated as **live capability providers**, not one-time scanned config entries.

#### 3) Policy and config filtering are first-class

The runtime explicitly uses logic such as:

- `filterMcpServersByPolicy`
- `doesEnterpriseMcpConfigExist`
- `isMcpServerDisabled`
- `setMcpServerEnabled`
- `dedupClaudeAiMcpServers`

So MCP enablement is not a raw “user added a URL” path. It is gated by:

- policy
- enterprise config
- enabled/disabled state
- source deduplication

#### 4) It is coupled to channel / relay logic

The same hook also references:

- channel notifications
- channel permission relay
- `registerElicitationHandler`

That suggests some MCP interactions are tied to approval flows, messaging channels, or relayed interactive behavior.

### Interpretation

This hook makes it clear that Claude Code treats MCP as a **hot-pluggable, policy-controlled, continuously synchronized capability network**.

---

## 3. `/mcp` is an operations surface, not a passive info page

`commands/mcp/mcp.tsx` supports at least:

- `/mcp`
- `/mcp reconnect <server>`
- `/mcp enable <server|all>`
- `/mcp disable <server|all>`
- `/mcp no-redirect`

### A revealing detail

It implements a dedicated `MCPToggle` component just so it can access `toggleMcpServer()` via React context.

This suggests the current MCP control surface is still fairly UI-context-centric rather than fully separated into a global service API.

### Another notable detail

For some build/user variants, base `/mcp` redirects into plugin management UI.

That implies the product already partially merges MCP management into the broader plugin/settings experience.

### Interpretation

`/mcp` is effectively an MCP operations entry point, not just a status display.

---

## 4. `MCPSettings`: full management UI, not hidden config

`components/mcp/MCPSettings.tsx` clarifies the product shape of MCP.

### What it does

- reads `mcp.clients` from AppState
- reads agent definitions and extracts agent-required MCP servers
- distinguishes transports:
  - `stdio`
  - `sse`
  - `http`
  - `claudeai-proxy`
- checks auth state for SSE / HTTP servers
- renders different menus and tool views by server type
- shows fallback guidance (`/doctor`, docs) when nothing is configured

### Interpretation

MCP is exposed as a real inventory and capability browser:

- server list
- transport type
- auth state
- tool list
- agent-required servers
- remote/proxy-backed servers

This is not a hidden developer-only subsystem.

---

## 5. `MCPReconnect`: the client state model is richer than online/offline

`MCPReconnect.tsx` is small but useful because it exposes the connection-state vocabulary.

Reconnect outcomes distinguish at least:

- `connected`
- `needs-auth`
- `pending`
- `failed`
- `disabled`

### Interpretation

Claude Code models MCP servers with a more nuanced lifecycle than simple online/offline:

- connected
- pending connection
- waiting for authentication
- failed
- disabled by configuration/policy

That aligns with the rest of the runtime and UI model.

---

## 6. MCP auth is heavy and clearly designed for real-world providers

`services/mcp/auth.ts` is one of the most complex files in the entire MCP stack.

### Immediately visible capabilities

It directly integrates with the MCP SDK auth stack for:

- auth discovery
- OAuth server discovery
- auth metadata
- token refresh
- OAuth client information
- token schemas

But it also adds a large amount of local behavior:

- localhost callback server
- PKCE / state / nonce handling
- secure storage
- lockfiles
- timeout / retry handling
- XAA (cross-app access)
- IdP / OIDC discovery and login
- XSS filtering
- safe logging with redaction
- OAuth error normalization

### Key observation 1: real-world provider weirdness is explicitly handled

The code contains special handling for:

- OAuth servers returning HTTP 200 with an error body
- Slack-style non-standard error codes such as:
  - `invalid_refresh_token`
  - `expired_refresh_token`
  - `token_expired`
- normalization into `invalid_grant`

That is a strong signal that the authors are handling actual provider edge cases, not just implementing RFC-happy paths.

### Key observation 2: sensitive parameter redaction is deliberate

The code explicitly redacts:

- `state`
- `nonce`
- `code_challenge`
- `code_verifier`
- `code`

This shows strong concern for log safety around OAuth flows.

### Key observation 3: secure storage + lockfiles imply multi-session reality

The auth layer uses secure storage and local locking, which suggests:

- tokens are persisted
- concurrent access matters
- multiple processes or sessions may touch the same auth state

### Key observation 4: MCP auth extends into enterprise identity territory

The file references:

- `performCrossAppAccess`
- `discoverOidc`
- `acquireIdpIdToken`
- `getXaaIdpSettings`
- `isXaaEnabled`

So the auth story goes beyond generic OAuth and into:

- OIDC
- enterprise identity providers
- cross-app access

### Interpretation

MCP auth should be treated as a substantial subsystem in its own right, not as a simple login helper.

---

## 7. `officialRegistry.ts`: evidence of an ecosystem layer

`officialRegistry.ts` fetches data from:

- `https://api.anthropic.com/mcp-registry/...`

and caches normalized official MCP URLs for lookup.

### Why this matters

Claude Code is not merely speaking the MCP protocol. It is beginning to distinguish:

- official MCP ecosystem entries
- non-official endpoints

That looks like the beginning of a curated ecosystem / marketplace layer.

---

## 8. `mcpStringUtils.ts`: MCP is deeply embedded in naming and permissions

This file contains utilities such as:

- `mcpInfoFromString`
- `getMcpPrefix`
- `buildMcpToolName`
- `getToolNameForPermissionCheck`
- `getMcpDisplayName`

### Why this is important

It shows that MCP tools are not treated as loose aliases. They are integrated into a standardized internal naming scheme:

- `mcp__server__tool`

That directly supports:

- avoiding collisions with built-in tool names
- precise permission rules scoped to:
  - a specific MCP server
  - a specific MCP tool

### Interpretation

Inside Claude Code, MCP is a first-class object in the permission and identity model, not merely a display-layer concern.

---

## 9. WebSocket transport: MCP is not limited to stdio/SSE heritage

`mcpWebSocketTransport.ts` implements JSON-RPC over WebSocket transport.

### Properties

- supports both:
  - Bun native WebSocket
  - Node `ws`
- wraps them into the MCP SDK `Transport` interface
- validates incoming JSON-RPC messages via schema parsing
- centralizes error / close / message handling
- emits diagnostic logs on failures

### Interpretation

Claude Code is not limiting MCP to early common transports like stdio and SSE. It is clearly evolving toward a broader remote-transport model.

That also creates possible overlap with remote / bridge capabilities elsewhere in the codebase.

---

## 10. What MCP appears to be inside Claude Code

Taken together, these files suggest MCP plays several roles:

### A. Capability bus

MCP servers can inject:

- tools
- commands
- resources

### B. Controlled connection layer

Those capabilities are admitted or withheld via:

- config
- enable / disable state
- reconnect paths
- auth state
- policy filters
- official-registry checks

### C. Bridge to other product subsystems

MCP clearly interacts with:

- the permission system
- agent definitions / agent workflows
- plugin management
- channel relay / elicitation
- remote/session-ingress authentication
- telemetry / analytics

### D. Candidate ecosystem surface

Official registry support, Claude.ai proxy transport, enterprise config, and OAuth/OIDC support all point toward an ecosystem / managed-platform direction, not just raw protocol compatibility.

---

## 11. Architectural assessment

### Four properties stand out

#### 1) MCP is a first-class extension model

Tools, commands, and resources are all integrated into AppState and UI.

#### 2) MCP auth is substantial enough to be its own product subsystem

OAuth, OIDC, IdP support, XAA, secure storage, and error normalization all point to real-world complexity.

#### 3) MCP is governed, not merely connected

The runtime applies:

- enterprise policy
- enable/disable control
- auth gating
- registry awareness
- source deduplication and filtering

#### 4) MCP directly shapes what Claude Code can do

It influences:

- what tools agents can access
- how permission rules are named and matched
- what dynamic commands appear in the command layer
- how resources are browsed and interacted with

---

## 12. Recommended follow-up questions

The highest-value next areas to inspect would be:

1. `client.ts` for concrete client creation and reconnect behavior
2. `config.ts` for merge logic across local, project, enterprise, and Claude.ai config sources
3. the exact semantics and trust model of `claudeai-proxy` servers
4. how `registerElicitationHandler()` integrates with approval flows
5. where channel permission relay is actually exercised
6. how session-ingress auth tokens relate to MCP auth boundaries
7. the full refresh path for resource/tool/prompt list changed notifications

---

## One-sentence summary

**Claude Code’s MCP subsystem is best understood as a governed extension bus: it brings external capability providers into the tool, command, resource, permission, auth, and UI layers of the runtime, while continuously managing them under policy, identity, and product-level controls.**
