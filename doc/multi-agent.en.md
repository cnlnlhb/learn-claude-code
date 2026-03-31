# Multi-Agent Module Analysis

## Scope

This analysis primarily covers:

- `source/src/tools/AgentTool/AgentTool.tsx`
- `source/src/tools/AgentTool/forkSubagent.ts`
- `source/src/tools/shared/spawnMultiAgent.ts`
- `source/src/utils/swarm/inProcessRunner.ts`
- `source/src/utils/swarm/leaderPermissionBridge.ts`
- `source/src/utils/teammateMailbox.ts`
- `source/src/utils/teamDiscovery.ts`
- `source/src/utils/tasks.ts`

## Executive Summary

Claude Code’s multi-agent system is much more than “spawn another sub-process.” It is a full collaboration runtime.

From the recovered source, it supports at least these collaboration forms:

1. **One-shot subagent / AgentTool delegation**
2. **Background async agents**
3. **team / teammate / swarm** collaboration
4. **in-process teammates** with context isolation
5. **tmux / pane-backed teammates**
6. **worktree isolation**
7. **remote isolation / remote tasks**
8. **fork subagents** that inherit parent context
9. **task lists + mailbox messaging + leader-side permission bridging**

So this is not merely “multi-agent support.” It is a serious attempt to make coordinated multi-agent work a first-class runtime model.

---

## 1. `AgentTool` is the control-plane entry point

### What the schema already reveals

The `AgentTool` input schema exposes a lot:

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

That means agent launches can vary by:

- task intent
- agent type
- model
- background execution
- team membership
- permission mode
- isolation strategy
- working directory

### Multiple output paths

The output schema and nearby internal types reveal multiple result states:

- `completed`
- `async_launched`
- internal states including:
  - `teammate_spawned`
  - `remote_launched`

So AgentTool is not a single execution path. It can return:

- synchronous completion
- async launch metadata
- teammate launch metadata
- remote-task launch metadata

### Interpretation

`AgentTool` is better understood as a delegation control plane than as a simple tool wrapper.

---

## 2. Multi-agent is deeply integrated, not a bolt-on experiment

The imports and dependencies in `AgentTool.tsx` show that multi-agent execution is woven into:

- analytics / GrowthBook
- LocalAgentTask / RemoteAgentTask
- worktrees
- remote teleportation
- permissions
- task-output storage
- SDK event queues
- agent summaries
- MCP requirement filtering
- agent swarms

That means multi-agent execution spans:

- UI
- tools
- task management
- permission handling
- remote execution
- telemetry

This is clearly core product architecture, not an isolated feature flag.

---

## 3. Fork subagents: context-inheriting branch workers

`forkSubagent.ts` exposes a distinct model from ordinary specialized subagents.

### Key properties

When fork is enabled:

- `subagent_type` becomes optional
- omitting it triggers an **implicit fork**
- the child inherits:
  - parent conversation context
  - parent system prompt
  - parent tool pool
  - parent model (`model: inherit`)
- permission mode becomes `bubble`
  - meaning permission prompts surface back to the parent terminal

### Prompt-cache optimization is deliberate

`buildForkedMessages()` is especially revealing:

- it preserves the full parent assistant message
- it creates identical placeholder tool-results for every tool-use block
- only the final directive text differs per fork child

That is an explicit attempt to keep the request prefix **byte-identical** across forked workers, maximizing prompt-cache reuse.

### Recursion guard

`isInForkChild()` prevents fork children from recursively forking again by detecting a special boilerplate tag in history.

### Interpretation

Fork is not a toy feature. It has clearly been designed with:

- cache efficiency
- recursion safety
- permission bubbling
- system-prompt inheritance

in mind.

---

## 4. In-process teammates: isolated agents inside the same process

`utils/swarm/inProcessRunner.ts` is one of the most interesting pieces in the whole architecture.

### What it does

The file comment states that it provides:

- AsyncLocalStorage-style context isolation
- progress tracking
- AppState updates
- idle notification back to the leader
- plan-mode approval flow
- cleanup on completion or abort

### What that means

Not every worker must run in its own process or tmux pane.

The runtime can also:

- simulate isolated teammates **inside the same process**

That usually means:

- lower startup overhead
- faster task turnover
- easier sharing of some in-memory runtime infrastructure
- but also a higher risk of context leakage if isolation is wrong

The code addresses that with teammate contexts, agent contexts, isolated task state, mailbox plumbing, and permission-routing machinery.

### Interpretation

Claude Code does not treat “multi-agent” as synonymous with “multi-process.” It supports both.

---

## 5. Leader-side permission bridge: workers act, leader approves

`leaderPermissionBridge.ts` is tiny but strategically important.

Its job is to:

- expose the REPL’s `ToolUseConfirm` queue setter to non-React code
- expose the leader’s permission-context setter
- allow in-process teammates to route permission requests through the leader’s normal approval UI

### How it plays out in `inProcessRunner.ts`

When a teammate tries to use a tool:

- explicit allow/deny decisions pass through immediately
- if the result is `ask`:
  - classifier auto-approval may be attempted first (for Bash, etc.)
  - otherwise the request is pushed into the leader’s `ToolUseConfirm` dialog

### Interpretation

The design is not “every worker opens its own permission prompts.” Instead, it tries to centralize authority:

- **workers execute**
- **leader approves**

That is a very human-friendly collaboration model because it preserves a single approval surface.

---

## 6. Mailbox: file-based inter-agent messaging

`teammateMailbox.ts` shows that swarm communication is not purely in-memory or WebSocket-based. There is also a file-backed messaging layer.

### Structure

Each teammate has an inbox file at roughly:

- `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

Other teammates can append messages to it.

### Properties

- file-backed
- lockfile-protected for concurrent writers
- stores:
  - sender
  - text
  - timestamp
  - read state
  - color
  - short summary

### Why this matters

This provides a communication substrate that is:

- persisted
- recoverable
- cross-process
- backend-agnostic

That is especially useful when the swarm includes a mix of:

- tmux teammates
- in-process teammates
- leader / worker combinations

---

## 7. Tasks are part of the collaboration skeleton: `tasks.ts`

This file shows the other half of collaboration: **shared task state**.

### Supported concepts

- task list IDs
- `pending` / `in_progress` / `completed`
- dependency edges via `blocks` / `blockedBy`
- high-water-mark task IDs
- concurrent mutation locks
- team-scoped task lists
- session fallback

### A critical detail

`getTaskListId()` resolves task ownership in this priority order:

1. explicit task-list ID
2. in-process teammate uses leader’s team name
3. process-based teammate uses `CLAUDE_CODE_TEAM_NAME`
4. leader-created team name
5. fallback to session ID

### Interpretation

Whether the agent is:

- in-process
- tmux/pane-backed
- standalone

it tries to converge on a **shared task view** whenever possible.

So multi-agent coordination is not mailbox-only. It uses:

- mailbox for communication
- task lists for shared work state

Together they form the collaboration backbone.

---

## 8. `spawnMultiAgent`: backend selection and runtime adaptation

`spawnMultiAgent.ts` is the real teammate-launch assembly layer.

### What it visibly handles

- model resolution and inheritance
- propagation of CLI flags
- propagation of settings / plugins / chrome / model flags
- backend detection:
  - tmux
  - iTerm2 / pane backend
  - in-process backend
- swarm session setup
- team-file writes
- teammate color assignment
- pane / split-pane creation
- mailbox writes
- task registration

### Especially notable details

#### Permission modes propagate

Modes such as:

- `bypassPermissions`
- `acceptEdits`
- `auto`

can be propagated to teammates via spawn config and inherited CLI flags.

But if `planModeRequired` is set, bypass permissions are intentionally not inherited.

That is strong evidence that the multi-agent stack and the permission stack were designed together.

#### Model inheritance exists for a reason

`resolveTeammateModel()` explicitly supports `inherit`, which suggests the system wants workers to stay aligned with the leader’s model unless there is a good reason not to.

### Interpretation

The spawn layer is not “just exec a command.” It is a runtime adaptation layer.

---

## 9. `teamDiscovery`: teams are persistent observable entities

`teamDiscovery.ts` reads team files and returns:

- team summaries
- teammate statuses
- running / idle state
- color
- cwd
- worktree path
- backend type
- permission mode

### What that implies

A team is not a one-off launch artifact. It is a persistent, inspectable entity that can be rendered and revisited.

That is fundamentally different from plain “spawn a subprocess once and forget it.”

---

## 10. What the multi-agent architecture seems to be

Taken together, these files suggest a structure like this:

### A. Delegation entry points

- `AgentTool`
- Team / Task / SendMessage style tools

### B. Execution forms

- synchronous agent
- async background agent
- fork child
- in-process teammate
- tmux / pane teammate
- remote agent
- worktree-isolated agent

### C. Collaboration infrastructure

- shared task lists
- teammate mailboxes
- team files
- status / discovery views
- progress tracking

### D. Centralized control surfaces

- leader permission bridge
- permission bubbling
- shared task identity
- shared team identity

This means Claude Code’s multi-agent story is not just “several Claudes running in parallel.” It is a structured orchestration system with:

- organizational identity
- messaging
- task coordination
- permission proxying
- UI observability
- backend abstraction

---

## 11. Architectural assessment

### Four properties stand out

#### 1) Collaboration is a first-class product capability

The integration points strongly suggest that multi-agent work is a core runtime concern, not an afterthought.

#### 2) The system supports multiple execution backends under one abstraction

- in-process
- tmux / pane
- remote
- worktree-isolated
- background task

That implies deliberate orchestration-backend abstraction.

#### 3) The leader acts as the control center

Rather than every worker independently owning everything, the system tries to funnel control back to the leader via:

- permission bridges
- shared task lists
- mailbox routing
- permission bubbling

#### 4) Efficiency and reuse are clearly prioritized

The fork path’s byte-identical prefixes, inherited models, exact tool-pool reuse, and shared task identity all point toward active optimization of:

- prompt caching
- context consistency
- orchestration overhead

---

## 12. Recommended follow-up questions

The highest-value next areas to inspect would be:

1. `runAgent.ts` as the main execution loop
2. RemoteAgentTask session establishment and callback/reporting paths
3. the full schema and persistence model of team files
4. how SendMessage / TeamCreate / TaskCreate compose into larger workflows
5. whether in-process isolation fully prevents shared-state contamination
6. edge cases of permission bubbling in nested delegation
7. how worktree-based workers reconcile their commits back to the leader’s workspace

---

## One-sentence summary

**The multi-agent subsystem is effectively a collaboration operating system: delegation, isolation, messaging, task orchestration, permission mediation, and status visibility are all integrated into a single swarm runtime.**
