# UX Architecture

## 1. UX objective

The frontend should feel closer to a modern local-AI desktop application than to a graphical wrapper around command-line flags.

The interface borrows useful interaction patterns from LM Studio and Jan while preserving substantially deeper llama.cpp control.

Primary rule:

> The user should navigate by intent, not by flag name.

Flag names remain visible as technical metadata and searchable aliases.

## 2. Main shell

Desktop-first three-region layout:

```text
+------+------------------------------------------+-----------------------+
|      |                                          |                       |
| NAV  |                MAIN AREA                 |      INSPECTOR        |
|      |                                          |                       |
| Chat |                                          | context-sensitive     |
|Model |                                          | configuration         |
|Server|                                          |                       |
|Monit.|                                          | Basic                 |
| Logs |                                          | Advanced              |
|      |                                          | Expert                |
| Set. |                                          |                       |
+------+------------------------------------------+-----------------------+
```

### Left navigation

Persistent high-level tasks only:

- Chat
- Models
- Server
- Monitor
- Logs
- Settings

Do not add one navigation item per llama.cpp parameter family. Parameter families belong inside the contextual inspector or a dedicated settings page.

### Main area

The user's current task.

Examples:

- conversation
- model library
- server/router status
- performance charts
- log stream

### Right inspector

Contextual configuration for the currently selected object.

The inspector can be collapsed and should remember its width.

## 3. First-run flow

The first-run experience should not start with dozens of performance controls.

```text
Welcome
  |
  v
Connect llama.cpp
  |
  +-- source tree
  +-- build directory
  +-- llama-server executable
  |
  v
Probe runtime(s)
  |
  v
Choose default runtime
  |
  v
Add model location / model
  |
  v
Ready
```

### Step 1: Connect llama.cpp

Explain the three choices briefly.

Recommended/default path for source builders:

```text
Select llama.cpp source folder
```

The application then discovers likely builds.

### Step 2: Runtime results

Example:

```text
Detected runtimes

[Selected] build/bin/llama-server
           CUDA
           version/build ...
           CUDA0: NVIDIA ...

[ ]        build-vulkan/bin/llama-server
           Vulkan
           version/build ...

[ Add external build ]
[ Select executable ]
```

Do not infer quality rankings such as "best" unless the criteria are explicit.

### Step 3: Model library

Allow either:

- choose one or more model directories
- add an individual GGUF

Do not require moving/copying the user's files into application-owned storage.

## 4. Models screen

Purpose: manage models, load state, compatibility, and persistent model-specific configuration.

### Main area

Supports card or dense-list view.

Each model item should show only high-value information by default:

```text
Qwen...
model-file.gguf

27B   IQ4...   16.2 GiB   Vision

Loaded on CUDA runtime

[Unload]  [Configure]
```

Possible state indicators:

- unloaded
- loading
- loaded
- unloading
- incompatible
- load failed
- runtime unavailable

### Model inspector

Sections:

#### Basic

- model file
- alias/display name
- context size
- GPU layers/offload
- batch size
- multimodal projector when relevant

#### Memory / Compute

- devices
- split mode
- tensor split
- main GPU
- KV offload behavior
- model-specific overrides allowed by the selected runtime/router

#### Chat

- chat template
- template kwargs
- reasoning-related model options where applicable

#### Adapters

- LoRA
- scaled LoRA
- control vectors

#### Advanced

More specialized model-loading settings.

#### Expert

Tensor overrides, metadata overrides, and other low-level controls.

### Inheritance visibility

Inherited settings must look inherited.

Example:

```text
Flash Attention
[ Auto ]    Inherited from runtime defaults
            [Override]
```

Once overridden:

```text
Flash Attention
[ On ]      Model override
            [Use inherited value]
```

## 5. Chat screen

Purpose: test and use the active model while clearly separating request-level settings from load-time settings.

### Header

Show:

- selected model
- selected runtime
- loaded state
- current context usage if known
- quick model switcher

### Conversation area

Standard role-based chat with support for the features the runtime/model exposes.

Potential capabilities:

- text
- image/media attachment
- tool calls
- reasoning display/handling
- regenerate
- stop
- edit/continue where supported by application semantics

### Composer

Keep normal usage uncluttered.

Quick controls can include:

- attachment
- reasoning mode when meaningful
- tools toggle when meaningful
- send/stop

### Chat inspector

Contains request/session-level generation controls:

#### Basic

- temperature
- max generated tokens

#### Sampling

- top-k
- top-p
- min-p
- repeat/frequency/presence penalties
- sampler sequence

#### Advanced sampling

- dynamic temperature
- XTC
- DRY
- Mirostat
- other detected samplers

#### Output constraints

- grammar
- JSON schema
- schema file

Changing a request-only control must not incorrectly mark the server as needing restart.

## 6. Server screen

Purpose: manage the selected `llama-server` process/router and API-facing configuration.

### Status header

Example:

```text
Server                      RUNNING
http://127.0.0.1:8086
Runtime: CUDA build ...

[Stop] [Restart] [Copy endpoint]
```

### Main sections

#### Endpoint

- host/bind address
- port
- API prefix if supported
- local network exposure

#### Authentication / security

- API keys
- TLS settings
- CORS

Warnings should explain consequences without blocking valid configurations.

#### Concurrency

- slots / parallel sequences
- continuous batching
- server threads
- timeouts

#### Router

Visible only when supported:

- models directory
- preset file
- max simultaneously loaded models
- autoload
- router-specific policies

#### Cache / slot persistence

- prompt cache
- cache RAM
- cache reuse
- slot save path
- persistence-related settings

#### API capabilities

Read-only support indicators where appropriate:

- OpenAI-compatible APIs
- Anthropic-compatible API if runtime exposes it
- embeddings
- reranking
- metrics
- slots
- multimodal
- tools

### Server inspector vs page body

Do not bury all server settings in the inspector.

Rule:

- status, topology, running models, and major sections live in the main area
- configuration for the selected section appears in the inspector

This keeps the app usable on medium-width screens.

## 7. Monitor screen

Purpose: make runtime behavior visible, not configure it.

Candidate panels, only when data exists:

- model/runtime status
- active/queued requests
- slots
- prompt processing throughput
- generation throughput
- speculative acceptance statistics
- cache statistics
- runtime uptime
- device usage where a reliable source exists

Do not fabricate GPU memory/temperature data from unreliable heuristics.

If hardware telemetry requires a separate platform-specific provider, show its provenance explicitly.

### Charts

Charts should be optional and efficient.

Do not make high-frequency polling mandatory for normal server operation.

## 8. Logs screen

Purpose: diagnostics and observability.

Features:

- live stdout/stderr
- pause/resume view without stopping capture
- severity filter when parseable
- text search
- copy selection
- copy recent diagnostics
- save/export logs
- timestamps
- auto-scroll toggle
- raw mode

The application should preserve unparsed log lines.

## 9. Settings screen

Application settings only. Do not put model or request parameters here simply because they are "settings".

Sections:

### llama.cpp runtimes

- registered runtimes
- default runtime
- add source/build/executable
- reprobe
- edit display name
- remove frontend registration

Removing a runtime registration must not delete the actual llama.cpp installation.

### Model library

- watched/indexed model directories
- rescan behavior

### Appearance

- system/light/dark
- density if implemented

### Application behavior

- launch behavior
- restore last workspace
- update/check preferences if implemented later

### Diagnostics

- application version
- Qt version
- stored runtime probe summaries
- config/log locations

## 10. Progressive disclosure

Each parameter has a presentation tier:

- Basic
- Advanced
- Expert
- Experimental
- Deprecated

These tiers affect initial visibility, not capability.

### Basic

Frequently useful and understandable controls.

### Advanced

Performance/behavior controls that many llama.cpp users may tune.

### Expert

Low-level options where incorrect values are possible but still valid to expose.

### Experimental

Upstream experimental capabilities or frontend support not yet considered stable.

### Deprecated

Visible through search/import and optionally in the relevant section with a warning.

## 11. Global setting search

A command-palette-style setting search should be reachable from the UI and keyboard.

Search terms include:

- friendly labels
- semantic IDs
- upstream flags
- aliases
- descriptions
- category names

Example query:

```text
cache-type-k
```

Result:

```text
Model > Memory > KV cache > K type
Upstream: --cache-type-k / -ctk
Current: q8_0
Scope: model override
Supported by current runtime
```

Selecting the result navigates directly to the control and temporarily expands its section.

## 12. Control selection rules

Widgets are selected by semantics, not just primitive data type.

### Toggle

Use for true binary choices where auto/inherit is not meaningful.

### Tri-state / segmented choice

Use when a value can be Auto / On / Off.

### Combo box

Use for finite enum values.

### Slider + numeric editor

Use only when:

- the domain is meaningfully bounded
- intermediate values are useful
- precise text entry remains available

The numeric field is authoritative for exact entry.

### Logarithmic/preset-assisted slider

Appropriate for context-size-like values spanning several orders of magnitude.

### Path picker

Use for model, projector, grammar, certificate, cache, or preset files.

### Device allocator

Use a custom control for multi-device distribution rather than a comma-separated text box when the runtime semantics are known.

Always provide raw/expert editing where round-trip fidelity requires it.

## 13. Restart and apply semantics

A modified setting has an application behavior class:

- immediate frontend-only
- next request
- model reload required
- server restart required
- unknown

Show pending changes clearly.

Example:

```text
3 changes require server restart
[Review] [Restart and apply]
```

Do not restart automatically because a field changed.

## 14. Command preview

The effective startup command should be available contextually from Server/Model configuration.

Example presentation:

```text
Command preview

/path/to/llama-server \
  --model ... \
  --ctx-size ...

[Copy] [Export .sh] [Show inherited/default values]
```

Default preview should include only arguments that the configuration engine intends to explicitly pass.

A diagnostic mode may show effective defaults separately.

## 15. Raw arguments

There must always be a route for unsupported/new/custom arguments.

Suggested UI:

```text
Additional arguments

--new-upstream-feature value
--other-flag
```

But internally prefer a tokenized argument list over one opaque shell string.

When the user imports raw CLI text, parse it into tokens and known semantic values where possible.

## 16. Responsive behavior

Primary target is desktop, but the application should degrade gracefully on narrower windows.

At reduced width:

1. inspector becomes overlay/drawer
2. navigation can collapse to icons
3. main content remains primary

Do not attempt phone-style responsive design in v1.

## 17. Accessibility and clarity

- keyboard navigation
- visible focus state
- text labels in addition to ambiguous icons where space allows
- tooltips are supplementary, not the only explanation
- warnings must include text, not color alone
- numeric controls must be editable without mouse dragging

## 18. UX acceptance scenarios

### Scenario A: beginner

A user selects a llama.cpp runtime, adds a GGUF model, clicks Load, opens Chat, and sends a prompt without touching expert options.

### Scenario B: advanced single-GPU user

A user configures context, GPU layers, flash attention, K/V cache types, batch/uBatch, multimodal projector, and speculative mode without using raw arguments.

### Scenario C: expert CLI migration

A user imports an existing llama-server command. Known options populate rich controls; unknown options remain preserved and visible. Exporting reproduces equivalent arguments.

### Scenario D: multiple builds

A user registers CUDA and Vulkan builds from the same or different source trees and can switch runtimes while seeing unsupported settings revalidated.

### Scenario E: upstream adds a new flag

The selected runtime exposes an unknown option through `--help`. The frontend shows it as uncatalogued rather than hiding it until the next release.
