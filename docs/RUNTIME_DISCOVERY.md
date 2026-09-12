# Runtime Discovery and Capability Probing

## 1. Purpose

The frontend must identify the exact `llama-server` binary the user intends to run and determine what that binary actually supports.

The design deliberately separates three concepts:

1. llama.cpp source tree
2. build directory
3. runtime executable

They are related, but they are not interchangeable.

## 2. Why source-folder-only discovery is insufficient

CMake allows out-of-tree builds anywhere on disk. A source tree such as:

```text
/home/user/src/llama.cpp
```

may have builds at:

```text
/home/user/src/llama.cpp/build
/home/user/builds/llama-cuda
/mnt/fast/llama-vulkan
```

Therefore selecting the source root cannot guarantee discovery of all valid builds.

Likewise, a source tree may be newer than an existing compiled binary.

The executable is the authoritative runtime artifact.

## 3. First-run runtime setup

The setup wizard presents three entry paths.

### Option A: Select llama.cpp source folder

Best for users who clone and build llama.cpp themselves.

The frontend:

1. validates that the folder resembles a llama.cpp source tree
2. looks for likely in-tree build directories
3. validates candidate CMake build directories
4. detects candidate `llama-server` executables
5. probes each executable
6. presents the discovered runtimes
7. allows manually adding an external build directory

### Option B: Select build directory

Best for external/out-of-tree CMake builds.

The frontend:

1. inspects `CMakeCache.txt` when present
2. locates likely server executables
3. probes them
4. optionally derives the source root from CMake metadata

### Option C: Select llama-server executable

Best for downloaded/prebuilt binaries or unusual layouts.

Only the executable is required.

## 4. Candidate build discovery

When a source root is selected, do not recursively scan the entire filesystem.

Search should be bounded and deterministic.

### 4.1 Likely local paths

Candidate directories may include immediate children matching common build naming patterns:

```text
build
build-*
cmake-build-*
out
out-*
```

### 4.2 CMake validation

A directory containing `CMakeCache.txt` is a stronger candidate.

Where available, inspect values such as:

```text
CMAKE_HOME_DIRECTORY
CMAKE_BUILD_TYPE
GGML_CUDA
GGML_VULKAN
GGML_HIP
GGML_SYCL
```

The exact available variables may change upstream. CMake metadata is descriptive only and must not replace executable probing.

A CMake build should be associated with the selected source root only when its recorded source/home directory matches appropriately.

### 4.3 Executable candidates

Likely locations include:

```text
<build>/bin/llama-server
<build>/llama-server
```

Do not assume these are the only legal paths. Manual executable selection always remains available.

## 5. Runtime identity

Each runtime receives a stable frontend record.

Suggested structure:

```text
RuntimeRecord
  id
  displayName
  sourceRoot?
  buildRoot?
  executablePath
  versionText
  buildNumber?
  commitHash?
  backendHints[]
  devices[]
  capabilities
  probeTimestamp
  lastProbeStatus
```

The record should be keyed internally by a generated frontend ID rather than by the path alone, because paths can change.

## 6. Probe pipeline

Probing should be staged so failures are understandable.

### Stage 1: Basic executable validation

Check:

- file exists
- regular executable file
- user can execute it
- launch does not immediately fail due to missing dynamic libraries

### Stage 2: Version probe

Execute:

```text
llama-server --version
```

Capture stdout, stderr, exit status, and timeout condition.

Version information is advisory metadata. It does not replace capability probing.

### Stage 3: Device probe

Execute:

```text
llama-server --list-devices
```

Parse devices conservatively.

The raw output must also be retained for diagnostics in case the parser does not recognize a future format.

### Stage 4: CLI capability probe

Execute:

```text
llama-server --help
```

The parser extracts at minimum:

- canonical flag names
- aliases
- argument placeholders
- descriptions
- visible defaults where parseable
- visible enum choices where parseable
- environment variable mappings where exposed
- sections/groups where exposed

The raw `--help` output should be cached with a hash.

### Stage 5: Optional live API probe

Once the runtime is running, enrich capabilities with server endpoints and runtime properties.

Possible data sources include:

- health endpoint
- props endpoint
- models endpoint
- slots endpoint
- metrics endpoint

The exact set must be capability-checked rather than assumed.

## 7. Capability matching

The frontend contains a catalog of known semantic parameters.

Probe results are matched against that catalog.

Example:

```text
Detected CLI flag: --cache-type-k
       |
       v
Known semantic ID: memory.kv.type_k
       |
       v
Rich UI metadata
```

Aliases should resolve to one semantic parameter.

## 8. Unknown upstream options

A future llama.cpp version may expose options unknown to the frontend.

These must not disappear.

Unknown detected options should be listed under:

```text
Advanced
  -> New / uncatalogued upstream options
```

For an unknown option, the frontend can initially provide:

- exact flag
- raw description
- inferred value type when safe
- editable raw value
- enable/disable inclusion

Type inference must be conservative. If uncertain, use a raw text field rather than inventing a numeric or boolean interpretation.

## 9. Removed or renamed options

A known catalog parameter can be absent from the selected runtime.

Possible causes:

- older build
- renamed upstream option
- backend-specific availability
- compile-time feature difference
- upstream removal

The UI should show it as unsupported for that runtime, not silently substitute another flag unless the compatibility mapping explicitly knows the equivalence.

## 10. Runtime switching

Changing the selected runtime triggers:

1. capability refresh if cache is stale
2. configuration revalidation
3. visibility/enabled-state update for affected controls
4. preservation of semantic user choices where compatible
5. warnings for values that cannot be represented by the new runtime

No setting should be silently dropped.

## 11. Probe caching

A successful probe can be cached based on a fingerprint containing information such as:

- executable canonical path
- file size
- modification timestamp
- version output hash

If the executable changes, capabilities are reprobed.

The user can always force a re-probe.

## 12. Failure UX

Examples:

### Missing libraries

```text
Could not start llama-server

The executable exists but the dynamic loader failed.

[Show technical details]
```

### Executable is not llama-server

```text
This executable does not appear to expose the expected llama-server interface.
```

### Old runtime

```text
Runtime detected, but some frontend features are not supported by this llama.cpp build.
```

The application should still work with the supported subset when practical.

## 13. Security and execution rules

- Never interpolate the executable path into a shell command for probing.
- Use argument vectors.
- Do not source shell scripts automatically.
- Do not execute arbitrary files merely because they appear under a source/build tree; candidate filenames must match expected runtime executables and still pass validation.
- Preserve raw probe output for diagnostics, but avoid exposing secrets if future upstream output includes sensitive environment-derived values.

## 14. Acceptance criteria

Runtime discovery is considered complete for v1 when:

- a user can register a source tree and discover common in-tree builds
- a user can register an arbitrary external build directory
- a user can register a standalone `llama-server`
- multiple runtimes can coexist
- runtime version and devices are shown
- `--help` is parsed into capabilities
- unknown upstream options remain accessible
- switching runtime updates support/validation without losing user configuration silently
