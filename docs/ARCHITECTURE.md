# Application Architecture

## 1. Technology baseline

- Qt 6
- C++20
- QML / Qt Quick for presentation
- CMake
- Qt Network for HTTP/SSE/WebSocket needs where applicable
- QProcess for llama.cpp process management
- Qt Test for core/unit tests

The application should avoid embedding llama.cpp inference directly in v1. It manages and communicates with external llama.cpp runtimes.

## 2. Architectural rule

QML is a presentation layer.

It must not:

- construct llama.cpp CLI arguments directly
- parse `--help`
- decide compatibility
- own persistent runtime/model configuration
- launch processes itself

Those responsibilities belong to C++ services/models.

## 3. High-level components

```text
QML UI
  |
  v
View Models / Controllers
  |
  +------------------------------+
  |                              |
  v                              v
Configuration Engine        Runtime Manager
  |                              |
  v                              +--> Capability Probe
Parameter Catalog                +--> Process Supervisor
  |                              +--> API Client
  v                              +--> Metrics / Logs
Validator
  |
  v
Command Builder
```

## 4. Proposed modules

### `core/parameter`

Owns the semantic description of llama.cpp settings.

Classes/concepts:

- `ParameterId`
- `ParameterDefinition`
- `ParameterCatalog`
- `ParameterValue`
- `ParameterOrigin`
- `ParameterSupportState`
- `ParameterPresentationTier`

### `core/config`

Owns selected values and inheritance.

Classes/concepts:

- `ConfigurationLayer`
- `EffectiveConfiguration`
- `ConfigurationResolver`
- `ConfigValidator`
- `ValidationIssue`

### `core/runtime`

Owns registered llama.cpp runtimes and probing.

Classes/concepts:

- `RuntimeRecord`
- `RuntimeRegistry`
- `RuntimeProbe`
- `RuntimeCapabilities`
- `DeviceInfo`
- `HelpParser`

### `core/command`

Owns CLI serialization/deserialization.

Classes/concepts:

- `CommandBuilder`
- `CommandParser`
- `ArgumentToken`
- `RawArgument`

Output is executable path + argument vector, not shell text.

A separate renderer can produce human-readable shell syntax for preview/export.

### `server/process`

Owns process lifetime.

Classes/concepts:

- `ServerProcess`
- `ServerState`
- `ServerLaunchRequest`
- `ProcessOutputBuffer`

Responsibilities:

- launch
- graceful stop where possible
- terminate/kill fallback
- restart
- exit/crash detection
- stdout/stderr capture

### `server/api`

Owns communication with the running server.

Classes/concepts:

- `ServerApiClient`
- `HealthClient`
- `ModelsClient`
- `SlotsClient`
- `MetricsClient`
- `ChatClient`

Endpoint use must be capability-aware.

### `models/`

Frontend model library, distinct from llama.cpp model internals.

Classes/concepts:

- `ModelRecord`
- `ModelLibrary`
- `ModelMetadata`
- `ModelProfile`
- `ModelLoadState`

### `profiles/`

Persistent reusable configuration.

Classes/concepts:

- `RuntimeProfile`
- `ModelProfile`
- `RequestProfile`
- `ProfileStore`

Profiles should store user intent/overrides, not unnecessary snapshots of every upstream default.

### `telemetry/`

Read-only runtime statistics and optional hardware telemetry.

Do not make platform-specific GPU telemetry part of core server correctness.

### `ui/`

C++ view models exposed to QML.

Examples:

- `AppViewModel`
- `RuntimeViewModel`
- `ModelLibraryViewModel`
- `ServerViewModel`
- `ChatViewModel`
- `SettingsSearchModel`
- `ParameterSectionModel`

## 5. Parameter value representation

Avoid QVariant-as-everything inside the core domain where practical.

A conceptual value representation needs explicit special states:

```text
ParameterValue
  state:
    Unset
    Auto
    Enabled
    Disabled
    All
    Explicit
    Raw
  payload: typed value when applicable
```

Exact C++ representation can use `std::variant` or a purpose-built value class.

The important requirement is preserving the distinction between omitted, automatic, and explicitly selected values.

## 6. Configuration layers

Suggested layer model:

```text
Upstream defaults
      |
Frontend global defaults
      |
Runtime profile
      |
Model overrides
      |
Session/request overrides
      v
Effective configuration
```

Resolution must retain provenance.

Example:

```text
memory.kv.type_k = q8_0
origin = ModelOverride
```

QML can then display both value and source.

## 7. Parameter catalog format

The catalog should be data-driven but validated by C++ code.

A definition conceptually contains:

```json
{
  "id": "memory.kv.type_k",
  "cli": {
    "canonical": "--cache-type-k",
    "aliases": ["-ctk"]
  },
  "type": "enum",
  "scope": ["runtime", "model"],
  "presentation": {
    "category": "memory.kv",
    "tier": "advanced",
    "label": "KV cache K type",
    "control": "combo"
  },
  "compatibility": {},
  "validation": {},
  "apply": "model_reload"
}
```

Do not treat one static JSON file as infallible. The selected binary's probe result is combined with catalog metadata.

## 8. Catalog vs runtime capabilities

```text
Known catalog       Probed runtime
     |                    |
     +---------+----------+
               v
       Effective parameter model
```

Cases:

### Known + detected

Rich supported control.

### Known + not detected

Unsupported for current runtime.

### Unknown + detected

Generic uncatalogued control.

### Known deprecated

Still importable/searchable, with warning.

## 9. Command generation

The command builder receives:

- selected runtime executable
- resolved startup configuration
- raw preserved arguments

It emits:

```text
Executable: /path/to/llama-server
Arguments:
  ["--model", "/path/model.gguf", "--ctx-size", "200000", ...]
```

Validation occurs before launch.

The builder must have round-trip tests for quoting-sensitive paths even though `QProcess` avoids shell quoting during execution.

## 10. Command import

Importing an existing shell-form command is inherently more difficult than building an argument vector.

v1 parser should support common POSIX shell formatting:

- whitespace tokenization
- single quotes
- double quotes
- backslash escaping
- line continuations

It should not execute command substitutions, environment expansions, redirects, pipes, or arbitrary shell syntax.

Unsupported shell constructs should produce an explicit warning and preserve raw text for manual correction.

Security rule: command import is parsing, never execution.

## 11. Server lifecycle state machine

Suggested states:

```text
Stopped
  -> Starting
  -> Running
  -> Stopping
  -> Stopped

Starting -> Failed
Running  -> Crashed
Stopping -> FailedToStop
```

Additional transitional states can be added for router/model operations, but process state should remain explicit.

## 12. Pending configuration changes

Runtime configuration should distinguish:

- saved desired configuration
- configuration used for currently running process

This allows the UI to show:

```text
Desired: ctx = 200000
Running: ctx = 131072
Status: restart required
```

Never pretend a changed startup field is already active.

## 13. Persistence

Use a versioned frontend configuration format.

Candidate persistent areas:

```text
app settings
runtime registry
model library/index
profiles
desired server configuration
UI state
```

Do not persist secrets casually in plain project profile exports.

Authentication tokens/keys need a separately considered storage policy.

## 14. Logging

Keep application logs and llama-server logs conceptually separate.

- frontend log: app behavior, probes, parsing, API errors
- runtime log: captured llama-server stdout/stderr

Diagnostic export can combine them with explicit labels.

## 15. Error model

Errors should carry:

- machine-readable code
- concise user message
- technical details
- relevant runtime/path/operation metadata

Avoid using raw stderr as the only user-facing error.

## 16. Testing strategy

### Unit tests

Highest priority:

- help parser
- parameter catalog validation
- configuration inheritance
- validation rules
- command builder
- command import parser
- profile serialization

### Golden tests

Store representative `llama-server --help` fixtures from multiple upstream versions/builds.

Verify:

- known parameters are recognized
- aliases resolve correctly
- unknown parameters survive
- removed/renamed options are handled intentionally

### Integration tests

When a test runtime is available:

- probe binary
- launch server with minimal model/test setup where feasible
- health check
- stop cleanly

The core configuration pipeline should remain testable without a GPU.

## 17. Suggested source tree

```text
llamacpp-frontend/
  CMakeLists.txt
  src/
    main.cpp
    core/
      parameter/
      config/
      runtime/
      command/
    server/
      process/
      api/
    models/
    profiles/
    telemetry/
    ui/
  qml/
    shell/
    pages/
      Chat/
      Models/
      Server/
      Monitor/
      Logs/
      Settings/
    components/
    inspectors/
  data/
    parameters/
  tests/
    fixtures/
    unit/
    integration/
  docs/
```

## 18. Dependency direction

Keep dependencies one-way where possible:

```text
QML -> UI view models -> domain/core -> Qt/platform services
```

The core parameter/configuration logic should not depend on QML.

This is important because the parameter engine is the long-lived part of the application; the UI can evolve without rewriting configuration semantics.
