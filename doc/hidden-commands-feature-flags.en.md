# Hidden Commands & Feature Flags

[Back to README](../README.md) · [Documentation Index](./INDEX.en.md) · [中文版本](./hidden-commands-feature-flags.zh-CN.md)

This note summarizes what the recovered `@anthropic-ai/claude-code@2.1.88` package suggests about hidden commands, gated functionality, and public-build stubs.

---

## Executive summary

The public package clearly contains signs of internal-only or feature-gated functionality, but an important pattern shows up repeatedly:

- **many suspicious commands exist in the public tree only as hidden stubs**
- these stubs typically export something like:
  - `isEnabled: () => false`
  - `isHidden: true`
  - `name: 'stub'`

This strongly suggests the public package preserves command registration slots or build structure for internal/debug tooling, while disabling actual behavior in the external build.

So the right conclusion is not “all hidden commands are available,” but rather:

> the public package still leaks the shape of internal/debug capability surfaces even when implementation is stripped or gated off.

---

## 1. What counts as a “hidden command” here

In the recovered package, hidden functionality appears in at least three forms:

1. **command files present but stubbed**
2. **commands conditionally enabled by build flags / user type / env vars**
3. **capabilities registered only when feature gates are active**

This is different from a normal visible CLI command list.

---

## 2. Stubbed command files found in the public package

Several command paths that look highly internal/debug-oriented exist in the recovered tree as stub exports.

Examples observed:

- `source/src/commands/bughunter/index.js`
- `source/src/commands/mock-limits/index.js`
- `source/src/commands/oauth-refresh/index.js`
- `source/src/commands/ctx_viz/index.js`

The recovered content for these files is effectively:

```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

### Interpretation

This means:

- the public build still exposes the existence of these command names / slots
- but the actual implementation is removed or replaced
- they are intentionally hidden from normal command surfaces
- they are not usable in the public package as-is

### Why this matters

Even stub files reveal product structure.

From the names alone, these likely correspond to internal workflows such as:

- bug reproduction / debugging support (`bughunter`)
- mocked quota or rate-limit behavior (`mock-limits`)
- forced OAuth refresh tooling (`oauth-refresh`)
- context visualization / debug tooling (`ctx_viz`)

That is useful for reverse engineering the internal tooling surface, even without the underlying code.

---

## 3. Hidden/debug command names hinted elsewhere

Beyond the stub files above, earlier package scans also surfaced names such as:

- `ant-trace`
- `break-cache`
- `perf-issue`
- install-helper / app-install related command paths

Not all of these were confirmed as functional commands in the public build, but the naming pattern matters.

### Pattern

Many of these names look like:

- internal diagnostics
- performance investigation helpers
- cache invalidation tools
- auth/session maintenance tools
- support-oriented workflows

This is consistent with a mature internal operator/debug toolkit being partially represented in the shipped tree.

---

## 4. Build-time feature gating is pervasive

The package contains many code paths controlled by `feature('...')` checks from `bun:bundle`.

Examples visible in `main.tsx` / command-related code include:

- `feature('NEW_INIT')`
- `feature('COORDINATOR_MODE')`
- `feature('KAIROS')`
- `feature('TRANSCRIPT_CLASSIFIER')`

### Interpretation

This means the runtime is not simply one monolithic build with everything on.

Instead, it appears to support:

- dead-code-eliminated feature variants
- conditional imports
- functionality present only in certain builds
- hidden capabilities that may not even be bundled into external builds beyond placeholders

### Important consequence

When a command or subsystem appears in the source tree, that does **not** guarantee it is reachable at runtime in the public package.

---

## 5. `USER_TYPE === 'ant'` is an important boundary

Across the recovered package, many paths explicitly distinguish Anthropic-internal users from external users.

This shows up in places such as:

- GrowthBook env-var overrides
- prompt template selection
- internal logging behavior
- likely debug/admin tooling exposure

### Interpretation

The public package appears to preserve a split between:

- **internal / Anthropic-facing behaviors**
- **external / public behaviors**

That makes the stubbed hidden commands even more plausible as internal-only tools.

---

## 6. GrowthBook and remote config are another gate layer

Feature control does not appear to stop at build-time gating.

The recovered code also shows:

- GrowthBook-driven feature values
- remote-eval feature caches
- config overrides
- environment variable overrides
- auth-sensitive client reinitialization

### Why this matters

A capability may be:

1. compiled into some builds
2. visible in code
3. but still disabled by runtime gate/config

So there are multiple layers of visibility and availability:

- source-tree presence
- build inclusion
- command registration
- runtime feature gate
- user-type gating
- environment/config override

---

## 7. Hidden commands are only one part of the story — some “normal” commands are actually gated variants

The recovered `init` command is a good example.

In `commands/init.ts`, the prompt behavior switches between:

- an old init flow
- a newer multi-phase init flow

based on:

- `feature('NEW_INIT')`
- `process.env.USER_TYPE === 'ant'`
- `CLAUDE_CODE_NEW_INIT`

### Interpretation

So even a public command may have:

- external default behavior
- internal richer behavior
- override behavior via env var

That suggests command behavior itself is variant-gated, not just command existence.

---

## 8. `TRANSCRIPT_CLASSIFIER`, `KAIROS`, and `COORDINATOR_MODE` are especially revealing

These feature names matter because they imply larger hidden capability areas:

### `TRANSCRIPT_CLASSIFIER`
Likely controls classifier-based decision systems, including parts of auto mode / permission automation.

### `KAIROS`
Appears tied to assistant-mode behavior.

### `COORDINATOR_MODE`
Strongly suggests another orchestration or supervisory mode not necessarily exposed in the normal public UX.

### Interpretation

Even when not fully documented in the public package, feature names provide real clues about product direction and internal experimentation.

---

## 9. What we can responsibly conclude

### High confidence

- the public package contains hidden/stubbed command slots
- some internal/debug/admin-looking command names are preserved in the tree
- multiple feature layers exist: build-time, user-type, env-var, and GrowthBook/runtime gates
- external builds do not necessarily include or enable the same functionality as internal builds

### Medium confidence

- the stubbed command names correspond to real internal workflows/tools
- some richer modes (assistant/coordinator/classifier variants) are selectively exposed by build or gate

### Low confidence without further runtime testing

- which hidden commands are actually available in any public runtime path
- whether any stubbed command names remain invocable through non-obvious entrypoints
- which internal builds include full implementations for each stubbed command

---

## 10. Why this matters for repo readers

If you are using this research repo to understand Claude Code, this hidden-command pattern changes how you should read the source:

- do not assume every file in `source/src/commands/` is usable in the public CLI
- do treat stub files and feature names as clues about product structure
- do pay attention to `feature(...)`, `USER_TYPE === 'ant'`, GrowthBook, and env-var checks
- do distinguish between:
  - present in source
  - bundled in build
  - enabled at runtime

That distinction is essential for accurate reverse engineering.

---

## Bottom line

The public `@anthropic-ai/claude-code@2.1.88` package leaks the outline of a broader internal/debug feature surface through stubbed hidden commands and layered feature gates. The most defensible reading is that the public build preserves traces of internal capability areas, while removing or disabling their actual implementations.
