# Telemetry Module Analysis

## Scope

This analysis primarily covers:

- `source/src/services/analytics/index.ts`
- `source/src/services/analytics/sink.ts`
- `source/src/services/analytics/datadog.ts`
- `source/src/services/analytics/firstPartyEventLogger.ts`
- `source/src/services/analytics/growthbook.ts`
- `source/src/services/analytics/metadata.ts`
- `source/src/services/analytics/config.ts`
- `source/src/utils/telemetry/events.ts`

## Executive Summary

Claude Code’s telemetry system is much more than “call logEvent() everywhere.” It is a layered analytics and runtime-observability foundation that includes:

1. **a unified event entry point**
2. **a sink routing layer**
3. **a Datadog path**
4. **a first-party event logging path**
5. **GrowthBook feature gates, dynamic config, and experiment exposure**
6. **metadata enrichment with explicit sensitive-data constraints**
7. **an OTel-style event path**
8. **sampling, kill switches, privacy-level gates, and provider gating**

So telemetry is not a sidecar utility. It is a fairly mature data, experimentation, and observability substrate.

---

## 1. `analytics/index.ts`: the public logging API is intentionally minimal and strict

This file is architecturally interesting:

- the public API is essentially `logEvent()` / `logEventAsync()` / `attachAnalyticsSink()`
- the module is intentionally dependency-free to avoid import cycles
- events are queued before a sink is attached
- once the sink is attached, the queue is drained asynchronously

### Interpretation

The authors clearly want the logging entry point to be:

- **stable**
- **safe to call early**
- **insulated from sink complexity**

### A very important design choice

The metadata typing is intentionally restrictive:

- default metadata is basically `boolean | number | undefined`
- strings require explicit type assertions that document manual verification
- `_PROTO_*` fields are handled specially so privileged payloads do not leak to general-access sinks

### Conclusion

Claude Code’s telemetry system shows sensitive-data awareness at the API boundary, not just in downstream exporters.

---

## 2. `sink.ts`: unified routing, sampling, and kill switches

`sink.ts` is the routing layer.

### What it does

- determines whether an event should be sampled
- checks whether Datadog is enabled by gate
- strips `_PROTO_*` fields before Datadog fanout
- keeps the full payload for the first-party sink
- provides `initializeAnalyticsGates()` and `initializeAnalyticsSink()`

### Most important observation

#### 1) Datadog and 1P are not treated as equivalent sinks

- Datadog is treated as a general-access backend
- the first-party sink is treated as privileged

That is exactly why `_PROTO_*` fields:

- are stripped before Datadog
- can remain available to the 1P path and be hoisted into privileged proto fields

#### 2) Kill switches exist

The routing layer explicitly checks sink kill logic, which means telemetry fanout can be disabled dynamically rather than being a hard-coded always-on flow.

#### 3) Sampling is centralized

Sampling is not left to every call site. It is applied uniformly at the sink layer via `shouldSampleEvent()`.

### Conclusion

Claude Code is not doing ad hoc scattered logging. It has distinct layers for:

- entry
- enrichment
- routing
- sinks
- sampling
- kill switches

---

## 3. Datadog: whitelist-driven, low-cardinality operational telemetry

`analytics/datadog.ts` reveals a lot of production discipline.

### Immediate properties

- Datadog endpoint and client token are compiled into the code
- there is a `DATADOG_ALLOWED_EVENTS` allowlist
- events only ship in production
- events only ship for first-party provider mode
- batching, flush interval, batch size, and network timeout are all explicit

### Key observation 1: Datadog is not a full event mirror

Only selected events are allowed, such as:

- OAuth success/failure
- tool use success/failure
- API success/failure
- voice toggles
- team-memory sync
- crashes / uncaught exceptions

That strongly suggests Datadog is used primarily for:

- operational visibility
- failure monitoring
- high-level product counters

rather than as the complete analytics warehouse.

### Key observation 2: cardinality control is taken seriously

The code deliberately reduces cardinality by:

- normalizing MCP tool names to `mcp`
- collapsing model names for external users into canonical short names / `other`
- truncating dev version identifiers
- transforming `status` into `http_status` and `http_status_range`
- limiting queryable tag fields through `TAG_FIELDS`

This is classic mature Datadog hygiene.

### Key observation 3: shutdown flush exists

`shutdownDatadog()` flushes remaining logs before exit. That is why bridge/logout/shutdown paths explicitly call it.

### Conclusion

In Claude Code, Datadog behaves like:

- a low-latency operational telemetry path
- tightly bounded by allowlists and cardinality controls

not a generic unrestricted event bus.

---

## 4. First-party event logging: the richer internal event path

`firstPartyEventLogger.ts` shows there is a second, heavier analytics path.

### Characteristics

- built on the OTel logs SDK
- uses `LoggerProvider + BatchLogRecordProcessor`
- exports through a custom `FirstPartyEventLoggingExporter`
- batch behavior is driven by GrowthBook dynamic config
- emits structured payloads containing core metadata, user metadata, and event metadata

### Most important observations

#### 1) Sampling config is dynamic

- `tengu_event_sampling_config`

So sampling is not merely a Datadog concern; it is part of broader analytics policy.

#### 2) Batch behavior is remotely configurable

- `tengu_1p_event_batch_config`

with fields such as:

- `scheduledDelayMillis`
- `maxExportBatchSize`
- `maxQueueSize`
- `skipAuth`
- `maxAttempts`
- `path`
- `baseUrl`

That makes the 1P logger a remotely tunable subsystem, not a fixed hard-coded uploader.

#### 3) GrowthBook exposures also flow into 1P logging

The file contains dedicated `logGrowthBookExperimentTo1P()` logic, which means experimentation and event logging are linked into one pipeline.

### Conclusion

If Datadog is the operational telemetry path, first-party event logging looks more like:

- the internal structured analytics path
- the source of richer product / experiment data

---

## 5. `growthbook.ts`: feature gates, dynamic config, and experiment exposure are part of telemetry

This file is huge and central.

### It manages far more than on/off flags

The recovered code includes:

- GrowthBook client lifecycle
- client recreation after auth changes
- remote-eval feature caches
- env overrides and config overrides
- exposure logging
- refresh listeners
- reinitialization promises
- stale-cache / disk-cache fallback
- user attributes for targeting

### Key observation 1: GrowthBook is more than a feature-flag SDK

It also controls:

- dynamic config distribution
- event sampling config
- batch/export config
- security gates
- experiment exposure recording

In other words, GrowthBook is part of Claude Code’s runtime control plane.

### Key observation 2: stale vs blocking semantics are explicit

The code repeatedly distinguishes:

- `CACHED_MAY_BE_STALE`
- `CACHED_OR_BLOCKING`

That shows deliberate balancing between:

- startup latency
- correctness of gate decisions
- post-auth consistency

Especially security-sensitive gates are designed to avoid relying on stale cache when correctness matters.

### Key observation 3: the override model is comprehensive

The code supports:

- env-var overrides (ant-only)
- config overrides
- remote eval
- disk-cache fallback

This is useful for:

- internal testing
- eval harnesses
- local debugging
- staged rollout control

### Conclusion

GrowthBook in Claude Code is not just an A/B testing SDK. It is one of the central runtime configuration systems.

---

## 6. `metadata.ts`: telemetry is heavily enriched, gated, and privacy-aware

This file shows another sign of maturity: most metadata is not passed raw by call sites. It is centrally enriched.

### It handles things such as

- session / parent-session / team / teammate context
- model / betas / provider / platform / WSL / distro
- interactive mode / clientType / kairos active
- git / repo remote hash / VCS
- auth / subscription information
- whether MCP tool details are allowed to be logged
- skill-name extraction
- tool-input truncation / depth limits / collection limits

### A particularly important detail

MCP details are not logged in full by default.

Detailed server/tool names only flow when certain conditions are satisfied, for example:

- local-agent / cowork style contexts
- Claude.ai proxy MCP
- MCP URLs that match the official registry

Otherwise the telemetry uses more generalized representations.

### Interpretation

This is clear evidence that the authors are balancing:

- diagnostic value
- privacy risk
- user-specific MCP configuration exposure

### Conclusion

`metadata.ts` shows a telemetry system that has already internalized privacy grading and observability tradeoffs.

---

## 7. `utils/telemetry/events.ts`: there is also an OTel-style third-party/standard event path

This file exposes `logOTelEvent()`.

### Characteristics

- emits via `eventLogger.emit()`
- automatically adds:
  - `event.name`
  - `event.timestamp`
  - `event.sequence`
  - `prompt.id`
  - workspace host paths (events only, not metrics)
- `OTEL_LOG_USER_PROMPTS` controls whether prompt contents are redacted

### Interpretation

Claude Code appears to have at least three event paths:

1. Datadog
2. first-party event logging
3. OTel / standardized event logger

These are not redundant copies. They likely serve different purposes:

- operational search and alerting
- internal analytics and experimentation
- standard enterprise / telemetry integrations

---

## 8. `config.ts`: provider mode and privacy level are global gating conditions

`analytics/config.ts` is short but strategically important.

### Analytics is disabled in these cases

- test environment
- Bedrock
- Vertex
- Foundry
- telemetry-disabled privacy modes

### Interpretation

Claude Code is not assuming telemetry behaves identically across all provider modes.

For:

- third-party providers
- high-privacy modes

analytics can be disabled at the global gate level.

That is an explicit product boundary.

---

## 9. What telemetry appears to do inside Claude Code

Taken together, telemetry serves at least four roles:

### A. Operational observability

- API success/failure
- tool use success/failure
- crashes, reconnects, bridge heartbeat issues, etc.

### B. Product experimentation and rollout

- GrowthBook gates
- dynamic config
- exposure logging
- refresh behavior

### C. Internal behavior analysis

- first-party structured event logging
- rich user/session/environment context

### D. Enterprise / standardized export

- OTel event logger
- prompt redaction controls
- workspace-path handling rules

That means telemetry is not a cosmetic add-on. It is part of Claude Code’s runtime governance and observation model.

---

## 10. Architectural assessment

### Four properties stand out

#### 1) Logging is constrained, not free-form

Strict metadata typing, PII marker types, `_PROTO_` stripping, and gated MCP/tool detail logging all show a strong bias toward controlled logging.

#### 2) Multiple backends are clearly stratified

- Datadog: operational, allowlisted, low-cardinality
- 1P: internal analytics, structured payloads, configurable batching
- OTel: standardized / export-oriented event path

#### 3) Experimentation and telemetry are deeply coupled

GrowthBook is not just feature flags; it also influences sampling, batching, experiment exposure, and runtime controls.

#### 4) The design reflects real production pain points

Cardinality control, stale-gate handling, shutdown flushes, and provider/privacy gating all suggest a system shaped by actual production incidents rather than demo-level instrumentation.

---

## 11. Recommended follow-up questions

The next high-value areas to inspect would be:

1. `firstPartyEventLoggingExporter.ts` for final upload shape and redaction behavior
2. `sinkKillswitch.ts` for remote disablement strategy
3. exact field sourcing and caching behavior in `getEventMetadata()`
4. hot paths that trigger GrowthBook exposure logging
5. the final exporter destination of the OTel event logger
6. the specific mappings of `_PROTO_*` fields into privileged schema columns
7. differential effects of provider mode and privacy level across the different telemetry paths

---

## One-sentence summary

**Claude Code’s telemetry subsystem is best understood as a constrained data-and-experimentation infrastructure: event logging, privacy controls, dynamic configuration, experiment rollout, and multi-backend export are all organized into a mature runtime observability layer.**
