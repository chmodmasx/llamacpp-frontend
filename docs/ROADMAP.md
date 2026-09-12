# Implementation Roadmap

## Phase 0 — Product specification

Status: current phase.

Deliverables:

- product scope
- UX architecture
- runtime discovery design
- application architecture
- parameter schema design
- initial upstream parameter inventory

Exit criteria:

- every major user flow has a defined owner screen
- configuration scopes are explicit
- runtime discovery behavior is unambiguous
- no major architecture decision depends on QML implementation details

## Phase 1 — Core project skeleton

Deliverables:

- Qt 6 / C++20 / CMake project
- minimal QML shell
- unit-test target
- CI build on Linux
- code formatting/static-analysis baseline

Exit criteria:

- application starts
- tests run in CI
- QML shell loads without embedding configuration logic

## Phase 2 — Runtime registry and probing

Deliverables:

- add source tree/build/executable
- runtime registry persistence
- `--version` probe
- `--list-devices` probe
- `--help` capture/parser
- capability cache
- runtime switcher

Exit criteria:

- multiple llama.cpp builds can be registered
- a selected runtime shows identity, devices, and detected flags
- changing executable invalidates stale capability cache
- unknown upstream options are represented

## Phase 3 — Parameter engine

Deliverables:

- semantic parameter IDs
- parameter catalog format
- typed parameter values
- inheritance/origin model
- support states
- validation framework
- application/restart classification

Exit criteria:

- effective values can be resolved independently of QML
- unsupported parameters are detectable
- overrides can be reverted without losing inherited values

## Phase 4 — Command builder and importer

Deliverables:

- startup command builder
- argument vector generation
- shell preview renderer
- conservative POSIX command importer
- preservation of unknown arguments
- export to `.sh`

Exit criteria:

- representative llama-server configurations round-trip without meaningful loss
- startup does not rely on a shell
- unsafe shell constructs are never executed during import

## Phase 5 — Runtime setup UX

Deliverables:

- first-run wizard
- source/build/executable selection
- discovered-build list
- runtime diagnostics
- Settings > llama.cpp runtimes

Exit criteria:

- new user can configure a runtime without manually typing paths to flags
- external/out-of-tree builds remain possible

## Phase 6 — Model library

Deliverables:

- model directories
- add GGUF in place
- model metadata record
- model list/card UI
- per-model profile
- model inspector

Exit criteria:

- user can add/select/configure local models without copying files unnecessarily
- model overrides are visually distinguishable from inherited settings

## Phase 7 — Server process management

Deliverables:

- `QProcess` supervisor
- launch/stop/restart state machine
- stdout/stderr capture
- desired-vs-running configuration
- pending restart detection

Exit criteria:

- server lifecycle is reliable
- failed startup provides useful diagnostics
- edits do not falsely appear active before restart

## Phase 8 — Server/API page

Deliverables:

- endpoint status
- host/port/auth/CORS/TLS controls where supported
- concurrency controls
- router controls
- cache/slot controls
- health/API capability detection

Exit criteria:

- user can manage common server behavior graphically
- unsupported capabilities are clearly identified

## Phase 9 — Chat and request configuration

Deliverables:

- chat UI
- streaming response handling
- request/session settings inspector
- sampling controls
- output constraints
- reasoning/tool/media affordances when supported

Exit criteria:

- user can test loaded model without another client
- request-level controls do not require server restart unnecessarily

## Phase 10 — Advanced llama.cpp coverage

Deliverables:

- compute/device configuration
- KV cache configuration
- RoPE/YaRN
- multimodal configuration
- LoRA/control vectors
- speculative decoding modes
- tensor/metadata overrides
- uncatalogued upstream settings UI

Exit criteria:

- the parameter inventory marks every current llama-server flag as catalogued, intentionally raw, deprecated, or not applicable

## Phase 11 — Monitor and diagnostics

Deliverables:

- health/status
- request/slot state
- throughput metrics
- speculative metrics
- cache metrics
- logs page
- diagnostic export

Exit criteria:

- monitoring uses real exposed data and labels provenance
- unparseable runtime output remains accessible

## Phase 12 — Packaging

Deliverables:

- AppImage packaging
- desktop entry/icon
- portable application configuration policy
- documented external llama.cpp runtime workflow

Exit criteria:

- frontend installs/runs without bundling one mandatory backend-specific llama.cpp build

## Work rules

### Do not build UI before core semantics exist

A QML control should bind to a semantic parameter model, not hard-code an upstream flag.

### Do not claim complete coverage without an inventory

"Complete llama.cpp support" requires a machine-checkable inventory comparing the selected/current upstream help output against the frontend catalog.

### Keep phases reviewable

Each phase should leave the repository in a buildable/testable state before moving to the next.

### Prefer compatibility over clever inference

When the frontend cannot safely infer a new upstream parameter's semantics, expose it generically and preserve it rather than guessing.
