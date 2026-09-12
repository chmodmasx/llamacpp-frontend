# llamacpp-frontend

Native Qt frontend and visual runtime manager for `llama.cpp`.

## Goal

Provide an intuitive graphical interface for `llama-server` without hiding the capabilities available from the command line. The application should support progressive disclosure: simple defaults for ordinary use, while preserving access to advanced, expert, experimental, and newly detected upstream options.

## Planned stack

- Qt 6
- C++20
- QML / Qt Quick
- CMake

## Design principles

- Simple by default, complete when expanded.
- The selected `llama-server` binary is the runtime source of truth.
- QML presents semantic settings; it does not own llama.cpp flags.
- Unknown upstream arguments must remain accessible rather than disappearing.
- Startup uses `QProcess` with an argument vector rather than shell execution.
- Configuration must preserve inheritance, support state, and restart/reload semantics.

## Planning documents

- [Product specification](docs/PRODUCT_SPEC.md)
- [UX architecture](docs/UX_ARCHITECTURE.md)
- [Application architecture](docs/ARCHITECTURE.md)
- [Runtime discovery and capability probing](docs/RUNTIME_DISCOVERY.md)
- [Parameter schema](docs/PARAMETER_SCHEMA.md)
- [Implementation roadmap](docs/ROADMAP.md)

## Current status

Product and UX architecture planning. No implementation should begin until the initial upstream `llama-server` parameter inventory and schema mapping are reviewed.
