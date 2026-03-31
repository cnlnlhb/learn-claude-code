# Remote / Bridge Module Analysis

## Scope

This analysis primarily covers:

- `source/src/bridge/bridgeMain.ts`
- `source/src/bridge/trustedDevice.ts`
- `source/src/remote/RemoteSessionManager.ts`
- `source/src/remote/SessionsWebSocket.ts`
- `source/src/server/createDirectConnectSession.ts`
- `source/src/server/directConnectManager.ts`
- `source/src/utils/sessionIngressAuth.ts`
- `source/src/utils/teleport/api.ts`

## Executive Summary

Claude Code’s remote / bridge capability is much more than “run something on a remote machine.” It is a full session-ingress architecture that includes:

1. **bridge workers / bridge poll loops**
2. **remote-session WebSocket subscriptions**
3. **direct-connect mode**
4. **session-ingress token injection and rotation**
5. **trusted-device / elevated authentication**
6. **remote permission-request control channels**
7. **teleport-style remote event delivery and Sessions API integration**
8. **integration with worktrees, multi-session spawn, and remote task execution**

So this is not just a remote shell. It is a remote execution plane with identity, session management, permission routing, reconnect behavior, observability, and multiple transport models.

---

## 1. `bridgeMain.ts`: the bridge loop is a real daemon runtime

`bridgeMain.ts` is one of the heaviest files in the entire codebase.

### What its imports and state already reveal

It directly integrates with:

- analytics / GrowthBook
- Datadog / event-logging shutdown
- bridge API clients
- token-refresh schedulers
- session spawners
- trusted-device tokens
- worktree management
- multi-session spawn gates
- capacity wakeups
- environment/session ID compatibility layers
- worker registration / work-secret logic

### What `runBridgeLoop()` is responsible for

The function visibly tracks at least:

- `activeSessions`
- `sessionStartTimes`
- `sessionWorkIds`
- `sessionCompatIds`
- `sessionIngressTokens`
- `sessionTimers`
- `completedWorkIds`
- `sessionWorktrees`
- `timedOutSessions`
- `titledSessions`

That makes it clear the bridge is not simply “poll for a job and run it.” It is managing a persistent fleet of remote sessions.

### Interpretation

The bridge worker is best understood as:

- **an execution agent attached to a remote control plane**

rather than a one-shot CLI mode.

It has to:

- poll for work
- manage session lifecycle
- heartbeat active sessions
- re-queue/reconnect on token expiry
- sleep at capacity and wake when slots free up
- track per-session worktrees, timers, titles, and ingress credentials

### Conclusion

`bridgeMain.ts` makes it clear that Claude Code’s bridge mode is a long-lived worker runtime.

---

## 2. The remote model is session-centric, not command-centric

In `bridgeMain.ts`, the primary abstraction is not “run command X” but session lifecycle.

### Signals in the code

- `SessionSpawner`
- `SessionHandle`
- `SessionDoneStatus`
- `initialSessionId`
- session-ID compatibility logic
- `reconnectSession`
- session timeout watchdogs
- session title management

### What that implies

Claude Code’s remote architecture is not:

- send one request
- get one response

It is:

- create a session
- bind identity and ingress credentials to it
- keep it alive over time
- allow reconnection, resumption, naming, and timeout governance

That is why remote assistant, remote task, and remote agent experiences are possible.

---

## 3. Trusted-device enrollment: elevated auth for remote control

`bridge/trustedDevice.ts` is extremely revealing because it shows remote bridge sessions are treated as a higher-trust security tier.

The inline comments explicitly state:

- bridge sessions are `SecurityTier=ELEVATED` server-side
- sending `X-Trusted-Device-Token` is gated on the CLI side
- trusted-device tokens are persisted to keychain / secure storage
- enrollment must happen soon after `/login`

### Core mechanisms

- `getTrustedDeviceToken()` decides whether a token should be sent
- `clearTrustedDeviceToken()` clears stale tokens during login/account transitions
- `enrollTrustedDevice()` calls `/auth/trusted_devices` and persists a device token

### Interpretation

Claude Code does not treat remote bridge operations as normal low-risk API calls.

It is not satisfied with:

- “user has an OAuth token”

It additionally wants:

- trusted-device enrollment
- keychain-backed persistence
- stale-token clearing on account switches
- freshness constraints tied to recent login

### Conclusion

Trusted-device handling strongly suggests the bridge model is closer to “register this machine as a trusted remote endpoint” than to ordinary API usage.

---

## 4. `RemoteSessionManager`: remote sessions have both message flow and control flow

`remote/RemoteSessionManager.ts` captures the higher-level remote session experience.

### It coordinates

- WebSocket subscription for remote-session output
- HTTP POST delivery of user messages
- permission request / response handling
- cancel requests
- reconnect / disconnect callbacks

### The most important point

Remote traffic is not just assistant text. It distinguishes:

- `SDKMessage`
- `control_request`
- `control_response`
- `control_cancel_request`

and especially supports:

- `can_use_tool`

inside `control_request`.

### Interpretation

This means a remote agent can still send tool-permission requests back to the local client for approval.

So remote execution is not equivalent to “all permissions are decided remotely.” It supports:

- **remote execution with local approval feedback loops**

That is one of the most important architectural properties of the entire remote system.

---

## 5. `SessionsWebSocket`: remote sessions are long-lived subscriptions with reconnect semantics

`remote/SessionsWebSocket.ts` fills in the lower-level transport behavior.

### Key features

- connects to `/v1/sessions/ws/{sessionId}/subscribe`
- authenticates via Authorization headers
- supports both Bun and Node `ws`
- has ping/pong keepalive
- includes reconnect logic
- distinguishes permanent close codes from transient failures
- gives limited retries for `4001 session not found`

### Interpretation

Remote sessions are designed as long-lived subscriptions:

- they stay connected
- they stay authenticated
- they reconnect when possible
- they understand semantic close codes
- they explicitly account for transient session-state disruptions such as compaction windows

This is a mature real-time client design, not an incidental socket wrapper.

---

## 6. `sessionIngressAuth.ts`: token injection is a serious cross-process design problem

`utils/sessionIngressAuth.ts` is very important because it explains how remote-session auth tokens enter the current process.

### Token source priority

1. `CLAUDE_CODE_SESSION_ACCESS_TOKEN` environment variable
2. `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`
3. well-known token file fallback

### Why this matters

This design explicitly accounts for multiple runtime scenarios:

- parent process injects a fresh env token
- secure-ish file-descriptor delivery
- subprocesses that cannot inherit the FD fall back to a shared well-known file

### Two especially important details

#### 1) Tokens can be updated in-process

`updateSessionIngressAuthToken()` rewrites the env var in-process, which means the bridge can inject fresh tokens after reconnection without restarting the worker.

#### 2) Session keys and JWTs use different auth headers

If the token begins with `sk-ant-sid`, auth uses:

- `Cookie: sessionKey=...`
- `X-Organization-Uuid`

Otherwise it uses:

- `Authorization: Bearer ...`

### Conclusion

Session-ingress auth is not “just a string token.” It is a cross-process, hot-swappable, multi-identity credential-injection path.

---

## 7. Direct connect: a second remote path besides hosted session infrastructure

`server/createDirectConnectSession.ts` and `server/directConnectManager.ts` reveal a separate remote-execution path:

- connect directly to a server URL
- `POST /sessions`
- receive `sessionId`, `wsUrl`, `workDir`
- communicate over streaming JSON + control messages on WebSocket

### Characteristics of direct connect

- does not necessarily rely on the full hosted Sessions API / bridge cloud path
- looks more like direct attachment to a remote code server
- still supports:
  - sending messages
  - permission requests
  - interrupts
  - control responses

### Interpretation

Claude Code’s remote architecture is not singular. At minimum it supports:

- bridge / CCR / Sessions API mode
- direct-connect mode

In other words:

- **managed remote sessions**
- **direct remote endpoints**

coexist in the same product.

---

## 8. `teleport/api.ts`: remote sessions are productized “code sessions,” not mere shells

`utils/teleport/api.ts` makes the product boundary much clearer.

### It deals with things like

- retrying Session API requests
- listing and transforming code sessions
- resolving org UUID
- requiring Claude.ai OAuth tokens
- repository / git-source / knowledge-base sources
- session context
- outcomes / branches / GitHub PRs
- BYOC beta markers

### Interpretation

Claude Code’s remote sessions are not just “remote terminals.” They are tied to higher-level product entities:

- repositories
- git revisions
- knowledge bases
- outcome branches
- session context
- remote code sessions

This means remote execution has been productized as a persistent code-session concept rather than left as raw terminal access.

---

## 9. The remote / bridge layer is tightly coupled to other subsystems

Taken together, the remote layer clearly intersects with:

- **authentication**
  - OAuth tokens
  - org UUID
  - trusted-device state
  - session-ingress tokens

- **permissions**
  - remote control requests
  - `can_use_tool` approval flow
  - local approval with remote execution

- **MCP / agents / sessions**
  - remote sessions can still host dynamic runtime capabilities
  - bridge workers manage multiple remote sessions and task-like workloads

- **multi-agent / worktrees**
  - bridge code directly interacts with session spawners and worktree creation/removal

- **telemetry / analytics**
  - lifecycle, heartbeat failure, token failure, reconnects, and shutdown behavior are instrumented

### Conclusion

The remote / bridge layer is not an isolated networking subsystem. It is one of the major convergence points for high-privilege runtime behavior across Claude Code.

---

## 10. Architectural assessment

### Four properties stand out

#### 1) Remote capability is session-centric, not command-centric

Bridge, remote WebSockets, teleport sessions, and direct connect all revolve around session lifecycle.

#### 2) The local client still retains control authority

Even when execution is remote, permission requests can flow back to the local UI/client for approval.

#### 3) Identity and device trust are central to the design

Trusted-device tokens, ingress tokens, org UUIDs, and OAuth credentials all indicate that remote bridge behavior is treated as a sensitive, strongly authenticated capability.

#### 4) The system already looks like a platform layer

Bridge workers, Session APIs, BYOC paths, direct connect, viewer-only sessions, and multi-session spawn all suggest this is no longer a narrow feature — it is becoming a platform substrate.

---

## 11. Recommended follow-up questions

The highest-value next areas to inspect would be:

1. `bridgeApi.ts` for the full remote-control API surface and error semantics
2. `sessionRunner.ts` for how local Claude processes are bound to bridge sessions
3. server-side protocol and trust boundaries for direct-connect mode
4. how trusted-device tokens are validated, expired, and rotated server-side
5. the exact HTTP protocol behind `sendEventToRemoteSession()`
6. how remote sessions interact with compaction, transcripts, and task systems
7. what gates actually expose multi-session / multi-environment behavior in external builds

---

## One-sentence summary

**Claude Code’s remote / bridge subsystem is best understood as a high-privilege remote session plane: it combines session lifecycle management, strong identity, trusted-device auth, permission return paths, reconnect behavior, and multiple transport modes into a persistent remote execution system.**
