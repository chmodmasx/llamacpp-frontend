# Product Specification

## 1. Product definition

`llamacpp-frontend` is a native Linux desktop frontend and runtime manager for `llama.cpp`, focused first on `llama-server`.

The product is not merely a launcher. It should provide a visual, understandable representation of the capabilities exposed by the installed `llama-server`, while preserving the degree of control available from the command line.

The core promise is:

> Simple by default, complete when expanded.

A beginner should be able to load a model and run it without understanding every flag. An expert should be able to reach every supported setting without leaving the application.

## 2. Initial scope

### In scope for v1

- Linux desktop application.
- Qt 6 + C++20 + QML / Qt Quick.
- External `llama-server` runtime discovery and management.
- Multiple registered llama.cpp runtimes/builds.
- Model library for local GGUF files.
- Model loading/unloading through native llama.cpp capabilities.
- Complete visual configuration of `llama-server` startup settings.
- Request-level generation settings.
- Chat client for testing models.
- OpenAI-compatible server visibility and control.
- Runtime logs.
- Runtime metrics and health.
- Router / multi-model support where supported by the selected runtime.
- Multimodal configuration where supported.
- Speculative decoding configuration where supported.
- Import/export of profiles and CLI arguments.
- Capability probing against the actual selected binary.

### Deferred, but architecture must allow it

- `llama-cli` specific workflows.
- Quantization tools.
- imatrix workflows.
- perplexity / benchmark tools.
- Model conversion workflows.
- Automated llama.cpp compilation and backend installation.
- Windows/macOS packaging.

These are separate modules, not reasons to pollute the initial `llama-server` architecture.

## 3. Non-goals

- Reimplement llama.cpp inference inside the frontend.
- Fork llama.cpp unless absolutely necessary.
- Hide unsupported settings behind frontend-specific abstractions that cannot be exported.
- Maintain a permanent handwritten copy of every upstream flag as the only source of truth.
- Assume CUDA, a specific GPU vendor, a specific build layout, or a specific model family.
- Treat the source checkout as authoritative over the executable actually being run.

## 4. Product principles

### 4.1 Progressive disclosure

The UI presents commonly useful controls first, followed by Advanced, Expert, Experimental, and Deprecated sections.

No supported capability is intentionally removed merely to simplify the interface.

### 4.2 Binary is the runtime source of truth

The application may know about the llama.cpp source tree, git revision, and CMake cache, but runtime support is determined from the selected executable and its exposed capabilities.

### 4.3 Semantic configuration, not UI-bound flags

QML must never become the authoritative registry of command-line switches.

The UI talks to semantic parameter IDs such as:

- `context.size`
- `compute.gpu_layers`
- `memory.kv.type_k`
- `attention.flash`

A parameter catalog maps semantic IDs to version/build-specific CLI names, aliases, defaults, constraints, dependencies, and presentation metadata.

### 4.4 Round-tripability

Whenever possible:

- UI configuration -> valid llama.cpp command line
- imported llama.cpp command line -> UI configuration
- profile -> UI -> command line -> profile

Unknown CLI arguments must be preserved rather than silently discarded.

### 4.5 No shell execution for normal startup

`llama-server` should be launched with an executable path and argument vector via `QProcess`, not by constructing a shell command.

The shell-form command shown in the UI is a preview/export representation.

### 4.6 Explain settings in user terms

Each setting has:

- friendly name
- concise explanation
- exact upstream flag
- effective value
- default/origin
- scope
- restart behavior
- conflicts/dependencies
- optional expert notes

The user should not need to read `--help` to understand a normal setting.

## 5. Configuration scopes

Every setting belongs to one or more explicit scopes.

### 5.1 Application

Frontend-only behavior.

Examples:

- theme
- model library locations
- registered runtimes
- UI density

### 5.2 Runtime / engine

Settings that apply to the launched `llama-server` process or router.

Examples:

- host / port
- server authentication
- global thread settings
- router settings
- global logging

### 5.3 Model

Settings associated with loading a particular model.

Examples:

- context size
- GPU layers
- tensor split
- model-specific chat template
- LoRA configuration

### 5.4 Session / request

Settings that can vary without reloading the model where supported.

Examples:

- temperature
- top-k
- top-p
- penalties
- grammar / JSON schema
- maximum generated tokens

### 5.5 Effective-value hierarchy

The configuration engine must retain both the chosen value and its origin.

Suggested precedence:

1. upstream/runtime default
2. frontend global default
3. runtime profile
4. model override
5. session/request override

The UI must make overrides visible and allow reverting to inherited values.

## 6. Parameter state model

A setting is not always just a scalar.

The internal model must support states including:

- `unset` / inherit
- `auto`
- `enabled`
- `disabled`
- `all`
- explicit scalar value
- explicit list value
- structured value
- unknown raw value

Example:

`gpu_layers` may be `auto`, `all`, or an explicit integer.

This distinction must not be flattened into a single integer field.

## 7. Capability model

The application maintains a `RuntimeCapabilities` object produced by probing the selected runtime.

It includes at least:

- runtime version/build identifier
- supported command-line arguments
- accepted enum values where discoverable
- available devices
- server/API features
- multimodal support
- router support
- speculative decoding modes
- metrics support
- tool/MCP/agent support where exposed
- backend/build metadata where discoverable

Known capabilities receive a rich UI.

Unknown but discoverable options appear in an `Uncatalogued / New upstream options` section rather than disappearing.

## 8. Runtime registry

The application supports multiple llama.cpp runtimes.

Each registered runtime stores:

- display name
- source root, optional
- build root, optional
- executable path, required
- detected version/build
- backend/build metadata
- devices
- last probe result
- last successful use

Example runtimes might be CUDA, Vulkan, CPU-only, debug, or different llama.cpp revisions.

The application must not assume that one installation equals one build.

## 9. Model library

The model library should support:

- adding GGUF files without forcing a copy
- configurable model directories
- metadata extraction where practical
- multimodal projector association
- tags/favorites
- per-model profile
- runtime compatibility indication
- loaded/unloaded state

A model record should not duplicate runtime-global state unnecessarily.

## 10. Main application areas

### Chat

A practical client for testing the loaded model and request-level settings.

### Models

Model library, load/unload state, model configuration, runtime compatibility.

### Server

Runtime/router process configuration and API endpoint management.

### Monitor

Health, devices, memory where obtainable, throughput, slots, requests, cache/speculative statistics where exposed.

### Logs

Structured runtime output with filtering, search, copy, and diagnostics.

### Settings

Application-level configuration, registered llama.cpp runtimes, paths, appearance, and integration behavior.

## 11. Visual architecture

Desktop layout:

```text
+------+--------------------------------------+----------------------+
| NAV  |                                      | CONTEXT INSPECTOR    |
|      |              MAIN AREA               |                      |
| Chat |                                      | model/runtime/request|
|Model |                                      | settings             |
|Server|                                      |                      |
|Monit.|                                      | progressive sections |
| Logs |                                      |                      |
|      |                                      | command/effective    |
| Set. |                                      | value visibility     |
+------+--------------------------------------+----------------------+
```

The right inspector is contextual and can be collapsed.

## 12. Settings discoverability

The application requires a global setting search.

A search result must show:

- friendly name
- path/category
- upstream flag
- current value
- support status for current runtime

Searching for either `kv`, `cache-type-k`, or a friendly phrase should find the same setting.

## 13. Command preview and import/export

The application provides a live command preview representing the effective startup configuration.

Actions:

- copy command
- export shell script
- export frontend profile
- import command line
- reset parameter to inherited/default value

Unknown imported flags must survive export.

## 14. Validation philosophy

Validation has levels:

- Error: command cannot validly run as configured.
- Warning: likely invalid, conflicting, risky, or inefficient.
- Advisory: unusual but valid configuration.
- Information: useful effect explanation.

The frontend should not block unusual expert configurations merely because they are uncommon.

Validation rules should be data-driven where possible.

## 15. Compatibility philosophy

The frontend is allowed to understand upstream features better than the selected runtime, but it must never pretend unsupported features exist.

For each setting, UI states include:

- supported
- unsupported by selected runtime
- unknown support
- deprecated upstream
- detected but uncatalogued

Unsupported settings should normally remain discoverable in search, with a clear explanation of why they are unavailable.

## 16. Reference UX direction

The intended interaction model borrows useful concepts from LM Studio and Jan:

- compact sidebar navigation
- central task area
- contextual right-side settings inspector
- beginner-friendly defaults
- model-specific configuration
- dedicated server/developer view

It must not inherit their limitations where doing so would hide llama.cpp functionality.

Official references:

- https://lmstudio.ai/docs/app/user-interface/modes
- https://lmstudio.ai/docs/developer/core/server/settings
- https://www.jan.ai/docs/desktop/model-parameters
- https://www.jan.ai/docs/desktop/local-engine/llama-cpp

## 17. Definition of product success

The project succeeds when an advanced llama.cpp user can reproduce an existing CLI configuration through the GUI without losing meaningful settings, while a new user can load and use a compatible model without needing to understand the CLI first.
