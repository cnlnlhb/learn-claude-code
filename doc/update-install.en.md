# Update / Install Module Analysis

## Scope

This analysis primarily covers:

- `source/src/components/AutoUpdater.tsx`
- `source/src/utils/autoUpdater.ts`
- `source/src/utils/localInstaller.ts`
- `source/src/utils/nativeInstaller/index.ts`
- `source/src/utils/nativeInstaller/installer.ts`
- `source/src/utils/nativeInstaller/download.ts`
- `source/src/utils/nativeInstaller/packageManagers.ts`
- `source/src/commands/install.tsx`
- `source/src/commands/upgrade/upgrade.tsx`

## Executive Summary

Claude Code’s update / installation system is much more than a thin wrapper around `npm i -g`. It is a multi-path distribution stack that is clearly in the process of moving from a JS/npm installation model toward a **native installer** model. It includes at least:

1. **UI-driven auto-update checks**
2. **JS/npm global update paths**
3. **JS/npm local update paths**
4. **native installer paths**
5. **platform-specific binary / native package download flows**
6. **multi-process locking, version retention, atomic activation, and shell integration**
7. **installation-source detection** (brew / winget / deb / rpm / pacman / asdf / mise / npm)
8. **minimum-version gates, maximum-version kill switches, and update-channel management**

So “install and upgrade” is not an auxiliary script in Claude Code. It is already a distribution subsystem.

---

## 1. `AutoUpdater.tsx`: updates are a productized UI state machine

`AutoUpdater.tsx` makes the product shape of auto-updates very explicit.

### What it does

- checks for updates immediately on mount
- checks again every 30 minutes
- fetches current and latest versions
- queries server-driven `maxVersion`
- chooses update behavior based on actual installation type
- maps outcomes into UI state and telemetry events

### A key point

It does not assume a single update path. It first calls:

- `getCurrentInstallationType()`

and then branches among:

- `npm-local`
- `npm-global`
- `native`
- `development`
- unknown fallback

### Interpretation

Claude Code is clearly in a phase where **multiple installation origins coexist**, so the auto-updater must first identify how the current binary was installed before deciding how to update it.

### Another important detail

If the current install is not native, it first calls:

- `removeInstalledSymlink()`

That implies the old JS updater path and the newer native installer path can interfere at the launcher/symlink layer, so migration cleanup is built directly into the update flow.

---

## 2. `autoUpdater.ts`: JS update flow plus server-side version gates

This file is the core of the JS-based update path.

### Responsibilities include

- `assertMinVersion()` for hard minimum-version enforcement
- `getMaxVersion()` for maximum-version kill switches
- `getMaxVersionMessage()` for incident messaging
- `shouldSkipVersion()` for downgrade prevention via `minimumVersion`
- lock-file handling
- latest-version lookup
- npm/global install behavior

### Key observation 1: `minVersion` is a hard blocker

If the current version falls below `tengu_version_config.minVersion`, the CLI prints a mandatory upgrade message instructing the user to run:

- `claude update`

That means version obsolescence is controlled server-side, not left to release notes.

### Key observation 2: `maxVersion` is a server-side kill switch

If the newest available version is above the configured allowed `maxVersion`, the updater caps the upgrade target or skips updating entirely.

### Why this matters

The distribution system is designed not only to move forward, but also to:

- **stop rollout during incidents**
- **prevent bad versions from spreading further**

### Key observation 3: update locking is handled seriously

This file includes:

- `.update.lock`
- stale-lock takeover logic
- TOCTOU commentary
- explicit ENOENT / EEXIST race handling

That strongly suggests the authors expect real concurrent update attempts from multiple processes.

### Conclusion

`autoUpdater.ts` already exhibits the traits of a mature client-updater control plane:

- version gating
- rollout kill switches
- concurrency safety

---

## 3. `localInstaller.ts`: a transitional path to avoid npm-global permission problems

`localInstaller.ts` reflects a very practical reality:

- `npm -g` is not always reliable or permission-safe on user systems

### What it does

- creates a private install directory under `~/.claude/local`
- writes a local `package.json`
- writes a wrapper script named `claude`
- installs the Claude package locally with npm
- marks `installMethod: 'local'` on success

### Interpretation

This is a compromise path:

- it still uses npm package distribution
- but it avoids global-directory permission issues, sudo requirements, and some environment pollution

### Conclusion

The local installer looks like a stabilization / migration path that sits between legacy npm-global installs and the full native installer model.

---

## 4. `nativeInstaller/installer.ts`: this is the most client-like distribution path

`nativeInstaller/installer.ts` is one of the heaviest files in the whole recovered package.

### The file comments already summarize its ambitions

- directory structure management
- symlink activation
- multi-process safety
- fallback mechanisms
- support for both JS and native builds

### The storage model is very deliberate

It separates:

- data: installed versions
- cache: staging/download area
- state: locks
- user bin: executable entrypoint

That means the installer explicitly manages:

- durable installed versions
- temporary download/extraction state
- locking state
- the user-visible launcher path

### Core capabilities visible in the code

- platform-aware binary naming
- staged download followed by atomic move into install path
- version retention / cleanup
- stale-lock cleanup
- process-lifetime locking
- mtime-lock fallback
- shell integration / PATH repair / alias cleanup
- cleanup of legacy npm installs and symlinks

### A very important point

It is not just responsible for “placing a binary.” It covers the entire lifecycle:

- install
- activate
- lock
- clean up
- retain old versions
- repair shell integration
- migrate away from older install methods

### Conclusion

The native installer is one of the clearest signals that Claude Code is evolving into a more traditional client-distribution model rather than remaining purely an npm CLI.

---

## 5. `download.ts`: download sources are routed by user type

This file shows that distribution sources are not uniform.

### Two visible source families

#### For ant users

- Artifactory
- platform-specific native packages
- `npm view ... dist.integrity`
- `npm ci` with integrity verification

#### For external users

- GCS bucket
- `claude-code-releases`

### Key observation 1: direct version + stable/latest channels are both supported

`getLatestVersion()` accepts:

- explicit versions
- `stable`
- `latest`

It also gates special test-version patterns behind feature flags.

### Key observation 2: integrity verification matters

On the Artifactory path, the installer:

- fetches the platform package’s `dist.integrity`
- writes a lockfile
- relies on npm install/ci integrity validation

So the design is not “curl a binary and execute it.” It intentionally leans on package-manager integrity semantics where possible.

### Key observation 3: version checks are instrumented

The download path logs telemetry such as:

- `tengu_version_check_success`
- `tengu_version_check_failure`

which reinforces the point that the update path itself is treated as an observable production-critical system.

---

## 6. `packageManagers.ts`: install-source detection is a migration prerequisite

This file is easy to overlook but strategically important.

### It detects sources such as

- Homebrew
- winget
- pacman
- deb
- rpm
- apk
- mise
- asdf
- unknown

### Why that matters

Claude Code is clearly trying to unify / migrate multiple install origins. Without knowing whether the current instance came from:

- brew
- winget
- npm
- mise/asdf
- distro packages

it cannot safely decide:

- whether self-update should even run
- whether to suggest using the system package manager instead
- whether it is safe to clean up old installations

### Conclusion

Install-source detection is not cosmetic. It is foundational for compatibility and migration safety.

---

## 7. `/install`: native installation as a productized migration command

`commands/install.tsx` exposes the native install path as a first-class user command.

### Flow

1. determine target (`latest`, `stable`, or explicit version)
2. call `installLatest(channelOrVersion, force)`
3. run `checkInstall(true)` to repair launcher and shell integration
4. run:
   - `cleanupNpmInstallations()`
   - `cleanupShellAliases()`
5. persist settings like `autoUpdatesChannel`
6. present installation / setup / success / error state via UI

### Interpretation

`/install` is not just a downloader. It is a **migration command** that:

- installs the native build
- repairs the shell entrypoint
- removes legacy npm installs
- removes legacy aliases
- stores future update-channel preference

---

## 8. `/upgrade`: this is account-plan upgrade, not binary upgrade

`commands/upgrade/upgrade.tsx` is interesting because it does not update the program binary at all.

### What it does

- checks whether the current account is already on the highest Claude Max plan
- if not, opens:
  - `https://claude.ai/upgrade/max`
- then launches a new login flow

### Why this matters

In Claude Code, the word “upgrade” has two distinct meanings:

1. **software version upgrade** (install/update paths)
2. **account subscription upgrade** (Claude.ai Max path)

That is an interesting product choice: the same CLI mediates both client-distribution changes and account-capability upgrades.

---

## 9. What the update/install subsystem appears to do inside Claude Code

Taken together, it serves at least four roles:

### A. Distribution system

- version lookup
- channel selection
- package/binary download
- staged activation

### B. Migration system

- npm global → local/native migration
- shell alias cleanup
- launcher repair
- symlink management

### C. Compatibility system

- install-source detection
- multiple install-method coexistence
- platform-specific binary names and paths

### D. Risk-control system

- minimum-version enforcement
- maximum-version kill switches
- stale-lock / concurrent-update safety
- development-build auto-update suppression

That means install/update logic is not an auxiliary shell script. It is a proper client delivery layer.

---

## 10. Architectural assessment

### Four properties stand out

#### 1) The codebase is clearly in a migration from npm CLI distribution toward native client distribution

The coexistence of npm-global, npm-local, native installer, and legacy alias/symlink cleanup strongly suggests a transitional architecture.

#### 2) Installation and update behavior is already highly productized

UI state, install-source detection, settings persistence, shell integration, cleanup behavior, and telemetry are all wired together rather than left as ad hoc scripts.

#### 3) Concurrency and failure handling are relatively mature

Multiple lock mechanisms, stale-lock takeover, atomic moves, version retention, and min/max-version gating all indicate the authors are treating this as a real production distribution system.

#### 4) Account and client lifecycle are both mediated through the CLI

The CLI does not only manage binary version updates; it also mediates Claude.ai plan upgrades and re-login flows.

---

## 11. Recommended follow-up questions

The next high-value areas to inspect would be:

1. the full activation/rollback path inside `installLatest()`
2. signature/integrity boundaries after native binary download
3. how `checkInstall()` concretely edits PATH, aliases, and shell configs
4. how `cleanupNpmInstallations()` behaves across system package manager installs
5. whether native installer behavior may eventually conflict with direct package-manager ownership (brew/winget/deb/rpm)
6. how GCS and Artifactory release consistency is maintained
7. how version retention behaves under low-disk conditions

---

## One-sentence summary

**Claude Code’s update/install subsystem is best understood as a client-distribution and migration system: version control, download, installation, activation, concurrency safety, environment repair, and legacy-install cleanup are all integrated into one delivery layer.**
