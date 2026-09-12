# Parameter Schema

## 1. Purpose

The parameter schema is the contract between:

- upstream llama.cpp CLI/API capabilities
- runtime capability probing
- configuration inheritance
- validation
- command generation/import
- QML presentation

The schema must model user intent, not merely mirror strings from `llama-server --help`.

## 2. Core rule

A parameter has a stable frontend semantic ID independent of its current upstream spelling.

Example:

```text
Semantic ID: memory.kv.type_k
Current upstream CLI: --cache-type-k
Alias: -ctk
```

If upstream later renames an argument, the semantic ID can remain stable while compatibility mappings evolve.

## 3. Suggested definition shape

Illustrative JSON-like form:

```json
{
  "id": "memory.kv.type_k",
  "upstream": {
    "canonical": "--cache-type-k",
    "aliases": ["-ctk"],
    "environment": ["LLAMA_ARG_CACHE_TYPE_K"]
  },
  "value": {
    "kind": "enum",
    "states": ["unset", "explicit"],
    "allowed": [
      "f32",
      "f16",
      "bf16",
      "q8_0",
      "q4_0",
      "q4_1",
      "iq4_nl",
      "q5_0",
      "q5_1"
    ]
  },
  "scope": ["runtime", "model"],
  "apply": "model_reload",
  "presentation": {
    "page": "models",
    "section": "memory.kv",
    "tier": "advanced",
    "label": "KV cache K type",
    "control": "combo",
    "summary": "Precision used for KV-cache keys."
  },
  "validation": [],
  "relations": [],
  "compatibility": {}
}
```

The final serialization format may differ. Semantics are more important than JSON syntax.

## 4. Required fields

### `id`

Stable, unique semantic identifier.

Naming convention:

```text
<domain>.<subdomain>.<name>
```

Examples:

```text
model.path
model.alias
context.size
compute.gpu_layers
compute.devices
memory.kv.type_k
memory.kv.type_v
attention.flash
server.port
server.parallel
sampling.temperature
sampling.top_p
speculative.method
multimodal.projector.path
logging.verbosity
```

Do not encode current upstream flag spelling into the semantic ID unless it is genuinely the semantic concept.

## 5. Upstream mapping

Fields:

```text
canonical
aliases[]
environment[]
```

Optional compatibility metadata may map older/newer spellings.

Example:

```json
{
  "canonical": "--gpu-layers",
  "aliases": ["-ngl", "--n-gpu-layers"]
}
```

Aliases are not separate settings.

## 6. Value kinds

Minimum supported kinds:

### Boolean

True/false.

Use only when automatic/inherited state is represented separately.

### Integer

Examples:

- port
- thread count
- batch size

Metadata may define:

- min
- max
- step
- meaningful presets

Do not invent artificial maxima merely to enable a slider.

### Floating point

Examples:

- temperature
- top-p
- scale factors

Preserve sufficient precision for CLI round-trip.

### String

Free text.

### Enum

Finite known value set.

Allowed values may be augmented by runtime probe results if upstream exposes additional choices.

### Path

Subtype metadata:

- file
- directory
- executable

Optional filters are UI hints only.

### URL

String with URL-aware validation.

### List

Examples:

- devices
- API keys
- multiple LoRA adapters

List serialization is parameter-specific rather than assumed globally.

### Map / structured

For values naturally composed of keys/values or repeated structured entries.

Examples may include scaled adapters or typed metadata overrides.

### Raw

Fallback for uncatalogued or intentionally opaque upstream values.

## 7. Special value states

Primitive types are insufficient because llama.cpp uses semantic sentinel values and optional omission.

Supported states must include as applicable:

```text
Unset
Auto
Enabled
Disabled
All
Explicit
Raw
```

Additional parameter-specific symbolic states are permitted when justified.

### `Unset`

Frontend does not explicitly pass the setting at this layer.

The effective value may come from inheritance or upstream default.

### `Auto`

Frontend intentionally requests upstream automatic behavior when upstream exposes such a value.

`Unset` and `Auto` are not interchangeable.

### `All`

Symbolic upstream state, e.g. all GPU layers where supported.

Do not encode as an arbitrary large integer internally.

## 8. Defaults

Distinguish:

- frontend default
- upstream documented default
- runtime-detected/default-derived value
- model-derived default
- inherited value

A single `default` field is insufficient for all cases.

The UI should be able to say:

```text
Auto (llama.cpp default)
```

instead of pretending an inferred numeric value was explicitly selected.

## 9. Scope

Allowed scope identifiers:

```text
application
runtime
model
session
request
```

Some parameters may support more than one scope because router/model presets can override engine-level defaults.

The catalog must describe where the frontend permits overrides, separately from where upstream theoretically accepts an argument.

## 10. Apply behavior

Every parameter has an application class:

```text
frontend_immediate
request_immediate
model_reload
server_restart
startup_only
unknown
```

This drives pending-change UI.

A parameter can later gain a capability-specific apply strategy if upstream introduces hot reconfiguration.

## 11. Presentation metadata

Presentation is metadata, not core semantics.

Suggested fields:

```text
page
section
tier
label
shortLabel?
summary
details?
control
unit?
placeholder?
order
```

### Presentation tiers

```text
basic
advanced
expert
experimental
deprecated
```

### Control hints

Examples:

```text
toggle
tri_state
segmented
combo
spinbox
slider_spinbox
log_slider_spinbox
text
multiline
path_file
path_directory
device_selector
device_split_editor
key_value_editor
list_editor
raw_argument
```

QML may choose a more suitable adaptive component when appropriate, but must preserve semantics.

## 12. Explanations

A setting should have at least two explanatory layers.

### Summary

Short plain-language description shown inline or in tooltip/popover.

### Technical details

Optional deeper explanation including:

- upstream behavior
- performance/memory implications
- interaction with other settings
- caveats

Avoid claims such as "higher is better" where behavior is model/hardware dependent.

## 13. Relations

Parameters can depend on or conflict with others.

Relation types should include:

```text
requires
conflicts_with
enables
visible_when
relevant_when
mutually_exclusive_with
implies
```

Relations should reference semantic IDs, not QML object IDs.

Example concept:

```json
{
  "type": "relevant_when",
  "parameter": "compute.split_mode",
  "operator": "in",
  "value": ["layer", "row", "tensor"]
}
```

for tensor split configuration.

## 14. Validation

Validation rules can be:

- local scalar constraints
- cross-parameter constraints
- runtime-capability constraints
- filesystem constraints
- advisory heuristics

Severity:

```text
error
warning
advisory
info
```

Example:

```text
server.port must be 1..65535
```

is an error.

A potentially inefficient GPU split may only be advisory.

## 15. Compatibility

Compatibility metadata should avoid hard-coding only version-number ranges because upstream commits/builds can vary.

Prefer capability predicates such as:

```text
flag_detected("--cache-type-k")
enum_contains("tensor")
endpoint_detected("/metrics")
```

Version/build constraints can supplement capability detection where necessary.

## 16. Provenance

The effective setting exposed to UI should include:

```text
value
origin layer
explicit/inherited status
support status
running value when different
desired value
```

Example:

```text
Desired value: q8_0
Origin: model override
Running value: f16
Status: model reload required
```

## 17. CLI serialization

Each known parameter defines how it serializes.

Common patterns:

### Flag + value

```text
--ctx-size 200000
```

### Positive/negative flag pair

```text
--kv-offload
--no-kv-offload
```

### Optional enum

```text
--flash-attn auto
--flash-attn on
--flash-attn off
```

### Repeated/multi-value

Parameter-specific encoder.

### Omitted

`Unset` normally emits nothing.

The encoder should not assume all booleans use the same syntax.

## 18. CLI import matching

Import resolution order:

1. exact canonical flag
2. exact known alias
3. compatibility alias
4. unknown/raw argument

If two catalog definitions claim the same active alias, catalog validation fails.

## 19. Unknown argument schema

An uncatalogued detected argument gets a generated runtime-local definition containing at least:

```text
runtime flag
raw upstream description
argument placeholder if detected
inferred type confidence
value
include/exclude state
```

If type confidence is low, use raw string representation.

Generated unknown definitions must never be mistaken for stable semantic catalog IDs.

## 20. Catalog validation

At application build/test time, validate:

- unique semantic IDs
- no duplicate active canonical mappings
- aliases do not ambiguously collide
- referenced relation IDs exist
- enum defaults are valid
- numeric constraints are coherent
- presentation section/page identifiers exist
- apply behavior is specified

## 21. Inventory status

Each upstream argument in the coverage inventory should have one status:

```text
catalogued
generic_raw
not_applicable
deprecated
internal/help_only
needs_research
```

This allows a measurable coverage report instead of a subjective claim of completeness.

## 22. Coverage report

For a recorded upstream `llama-server --help` fixture, tests should be able to produce:

```text
Detected upstream options: N
Catalogued rich controls: N
Generic/raw controls: N
Known deprecated: N
Unclassified: N
```

Release policy should aim for zero unexplained/unclassified arguments for the upstream fixture targeted by that release.

## 23. Example: context size

Conceptual entry:

```json
{
  "id": "context.size",
  "upstream": {
    "canonical": "--ctx-size",
    "aliases": ["-c"],
    "environment": ["LLAMA_ARG_CTX_SIZE"]
  },
  "value": {
    "kind": "integer",
    "states": ["unset", "explicit"],
    "min": 0,
    "symbolic": {
      "0": "model_default"
    }
  },
  "scope": ["model"],
  "apply": "model_reload",
  "presentation": {
    "page": "models",
    "section": "context",
    "tier": "basic",
    "label": "Context size",
    "control": "log_slider_spinbox"
  }
}
```

The frontend may display `0` as `Model default`, but should retain knowledge of the upstream representation.

## 24. Example: Flash Attention

Conceptual entry:

```json
{
  "id": "attention.flash",
  "upstream": {
    "canonical": "--flash-attn",
    "aliases": ["-fa"]
  },
  "value": {
    "kind": "enum",
    "states": ["unset", "explicit"],
    "allowed": ["auto", "on", "off"]
  },
  "scope": ["runtime", "model"],
  "apply": "model_reload",
  "presentation": {
    "page": "models",
    "section": "attention",
    "tier": "basic",
    "label": "Flash Attention",
    "control": "segmented"
  }
}
```

## 25. Schema design acceptance criteria

The schema is sufficient for implementation when it can represent without special-case QML code:

- omitted vs automatic values
- aliases
- positive/negative flags
- enums
- bounded/unbounded numbers
- paths
- lists and structured entries
- inheritance
- support state
- restart/reload semantics
- dependencies/conflicts
- deprecated options
- unknown upstream options
- CLI serialization/import
