# Permissions & Risk Control Module Analysis

## Scope

This analysis primarily covers:

- `source/src/utils/permissions/PermissionMode.ts`
- `source/src/utils/permissions/permissions.ts`
- `source/src/utils/permissions/yoloClassifier.ts`
- `source/src/utils/permissions/bypassPermissionsKillswitch.ts`
- `source/src/utils/permissions/permissionsLoader.ts`
- `source/src/utils/permissions/denialTracking.ts`
- `source/src/tools/BashTool/bashSecurity.ts`

## Executive Summary

Claude Code’s permission system is not a single approval dialog. It is a layered execution-control stack that can be roughly divided into four levels:

1. **Mode layer** — decides the active permission posture of the session
2. **Rule layer** — performs static allow / deny / ask matching
3. **Classifier layer** — applies dynamic judgment via classifiers, auto mode, and YOLO-style risk evaluation
4. **Command-safety layer** — deeply inspects shell syntax for bypass and abuse patterns

This is better understood as a policy engine than a simple confirmation prompt.

---

## 1. Permission modes: `PermissionMode.ts`

### Visible modes

The recovered code clearly exposes these modes:

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto` (when `TRANSCRIPT_CLASSIFIER` is enabled)

That means the system is not a binary allow/deny switch. It supports multiple operating postures.

### Interpretation

- `default`: normal approval-oriented mode
- `plan`: planning-heavy / paused execution mode
- `acceptEdits`: more aggressive auto-accept posture for edits
- `bypassPermissions`: nearly disables human approval for many actions
- `dontAsk`: suppresses prompts even further
- `auto`: classifier-governed automation mode

### Important detail

`auto` is not treated as a standard external mode. The code explicitly treats it as special, tied to feature gates and internal/external distinctions.

This suggests permission modes are also part of product capability segmentation.

---

## 2. Rule engine: `permissions.ts`

This is one of the central files in the permission stack.

### Rule sources

The code models rules as originating from multiple sources, including:

- settings-backed sources
- `cliArg`
- `command`
- `session`

Together with `permissionsLoader.ts`, disk-backed sources include:

- `userSettings`
- `projectSettings`
- `localSettings`
- `policySettings`
- plus feature / managed-settings scenarios

### Rule behaviors

The core rule behaviors are:

- `allow`
- `deny`
- `ask`

### Matching sophistication

The rule engine is more than simple tool-name matching. It supports:

- whole-tool rules
- content-sensitive rules
- agent-type rules
- MCP server-level rules
- MCP tool-level rules
- split reasoning for multi-operation commands

For example, the code can determine:

- whether a whole tool matches a rule
- whether a specific agent type is denied
- whether an MCP server wildcard rule matches an entire server namespace

### Interpretation

This is already a multi-dimensional policy matcher, not a basic whitelist.

---

## 3. Approval prompts reveal the internal policy model

`createPermissionRequestMessage()` is surprisingly informative because it exposes the kinds of internal decision reasons that can force approval.

The system distinguishes reasons such as:

- classifier decision
- hook block
- matching permission rule
- subcommand decomposition with partial approvals required
- permission-prompt tool
- sandbox override
- working-directory reason
- safety check
- mode constraint
- async agent reason

This means “why approval is required” is not a single path. Multiple subsystems can independently trigger `ask` behavior:

- static rules
- dynamic classifiers
- hooks
- sandboxing
- working-directory constraints
- the active permission mode itself

This is a relatively explainable policy architecture, not a pure black box.

---

## 4. Managed policy can override local user rules: `permissionsLoader.ts`

One of the most important flags here is:

- `allowManagedPermissionRulesOnly`

If enabled, the runtime will only respect:

- `policySettings`

and ignore the usual broader set of permission-rule sources.

### Why this matters

This indicates explicit support for a managed / enterprise policy model where:

- local user allow/deny/ask rules may be suppressed
- organization-controlled policy becomes authoritative

The UI can also hide “always allow” style options:

- `shouldShowAlwaysAllowOptions()`

### Interpretation

This is not designed solely as a single-user local tool. It clearly anticipates:

- managed environments
- centrally controlled permissions
- conflicts between user autonomy and organizational policy

So the permission system is built for both **personal mode** and **managed mode**.

---

## 5. `bypassPermissions` and `auto` are not absolute: `bypassPermissionsKillswitch.ts`

This file is extremely important because it shows that:

- `bypassPermissions` is not always allowed
- `auto mode` is not always allowed either

### `bypassPermissions`

At startup, the code performs a gate check:

- if bypass mode is notionally available
- but remote / gate logic says it should be disabled
- the runtime rewrites the `toolPermissionContext` into a disabled-bypass variant

The check is also reset after login, implying that gate decisions depend on account / org identity.

### `auto mode`

`checkAndDisableAutoModeIfNeeded()`:

- verifies gate access based on current model / fast mode / permission context
- updates permission context if access is not allowed
- may enqueue a high-priority warning notification

### Interpretation

Even when the user tries to enter a more permissive mode, the system still reserves:

- remote capability gating
- org-level restrictions
- dynamic disablement / circuit breaking

So bypass / auto are best understood as **conditionally granted capabilities**, not raw local toggles.

---

## 6. Denial tracking: classifiers are not allowed to fail-closed forever

`denialTracking.ts` is short but revealing.

It tracks:

- `consecutiveDenials`
- `totalDenials`

with thresholds of:

- consecutive denials >= 3
- total denials >= 20

Beyond those thresholds:

- `shouldFallbackToPrompting(...)` becomes true

### Meaning

This means the classifier stack does not remain permanently fail-closed if it blocks too much. Instead, it falls back to explicit prompting.

That reflects a product tradeoff:

- if the classifier becomes too conservative
- and starts obstructing real work
- the runtime prefers human approval over endless denial

This is a usability safety valve.

---

## 7. YOLO / auto-mode classifier: `yoloClassifier.ts`

The naming is already revealing:

- YOLO classifier
- auto mode system prompt
- external permissions template
- Anthropic permissions template

### What the recovered code suggests

1. auto mode is backed by a prompt-driven classifier
2. the classifier prompt differs between:
   - external builds
   - Anthropic-internal builds
3. classifier requests / responses can be dumped locally
4. API failures can dump system prompt, user prompt, token-count comparisons, and action context
5. classifier-related diagnostics track:
   - main-loop token counts
   - classifier token estimates
   - transcript entry counts
   - model identity
   - action being classified

### Interpretation

Auto mode is not just a boolean feature flag. It is a real model-mediated decision subsystem with:

- prompt templates
- debug dump support
- context-comparison diagnostics
- rollout / feature-flag control
- external vs internal prompt variants

That is much closer to a secondary risk-assessment model than a simple setting.

---

## 8. Bash risk analysis is deep: `tools/BashTool/bashSecurity.ts`

This is one of the most technically serious parts of the entire risk-control stack.

### Immediate impression

The code is not relying on a few blacklist regexes. It is clearly written with shell-bypass techniques in mind.

### Explicitly handled attack surfaces include

#### Command / process / parameter substitution

- `$()`
- backticks
- `<()`
- `>()`
- `${}`
- `$[]`

#### Zsh-specific bypasses

- `=cmd` equals expansion
- `always {}`
- module-based abuse through `zmodload`
- `zpty`
- `ztcp`
- `sysopen / syswrite / sysread`
- builtin filesystem ops such as `zf_rm`, `zf_mv`, etc.

#### Heredoc + substitution edge cases

The file contains a dedicated `isSafeHeredoc()` path, with comments explicitly warning that an incorrect early-allow would bypass all later validators. The implementation therefore tries to be provably safe, not “probably safe.”

#### Additional controls

- malformed token injection
- IFS injection
- brace expansion
- control characters
- unicode whitespace
- backslash-escaped operators
- comment / quote desynchronization
- quoted newlines
- jq `system()` / file argument abuse
- `/proc` environ access
- tricky input/output redirection handling

### Interpretation

The threat model here is clear:

> attackers may exploit shell parsing differences, zsh-specific features, redirection tricks, quoting edge cases, and syntax ambiguities to bypass naïve safety checks.

So the Bash layer is treating the shell itself as an attack surface.

---

## 9. Important caveat: external builds may ship stubs

The recovered `bashClassifier.ts` in this package is an external-build stub:

- `isClassifierPermissionsEnabled() -> false`
- `classifyBashCommand(...)` returns disabled behavior

That means:

- not every distribution actually enables the full classifier-based permission feature set
- there may be real capability differences between external and internal builds

This is consistent with the other signals in the codebase:

- `USER_TYPE === 'ant'`
- feature gating
- separate external/internal prompt templates

So when analyzing permissions, we have to distinguish between:

- what the architecture supports in principle
- what this particular published package appears to enable in practice

---

## 10. Architectural assessment

### The most important properties of this design

#### 1) Multi-layered defense, not single-step approval

The recovered stack includes at least:

- permission modes
- rule matching
- classifier-based decisions
- shell-level command safety analysis
- hooks
- sandbox integration
- managed policy support

#### 2) Not purely user-local; clearly designed for governance

The codebase explicitly supports:

- managed permission rules only
- gated bypass / auto capabilities
- settings-source precedence and policy authority

#### 3) Not just preventing misclicks; actively resisting bypass techniques

The Bash safety layer especially shows awareness of:

- shell parser ambiguity
- zsh module abuse
- heredoc early-allow vulnerabilities
- quote/comment desynchronization
- disguised redirections

#### 4) Usability has a fallback path

Denial tracking ensures the system can fall back to prompting if classifiers become too obstructive.

This suggests the authors are trying to balance safety and usability dynamically, rather than always maximizing conservatism.

---

## 11. Recommended follow-up questions

The next high-value areas to inspect would be:

1. how `permissionSetup.ts` constructs disabled / gated permission contexts
2. the exact decision path inside `classifierDecision.js`
3. how `shouldUseSandbox()` interacts with rule-based decisions
4. how MCP tools are represented and permission-bounded
5. how async-agent permissions are scoped
6. how Bash safety results ultimately merge with allow/deny/ask policy
7. how external builds differ from Anthropic-internal builds in permission features

---

## One-sentence summary

**The permissions subsystem is best viewed as a layered execution-governance engine: modes, policy rules, classifiers, sandbox constraints, and shell-level exploit defenses all combine to decide whether a tool action is allowed, denied, or escalated for approval.**
