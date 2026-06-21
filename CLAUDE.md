# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Giada is a cross-platform (Linux/Windows/macOS) C++ desktop loop machine / sample player /
sequencer. The audio engine runs in a hard-realtime thread; the GUI is FLTK. Code is C++20,
built with CMake + vcpkg, dependencies vendored as git submodules under `src/deps/`.

## Build & test

Dependencies come from two places: third-party libs via **vcpkg** (manifest `vcpkg.json`, needs
`VCPKG_ROOT` set), and vendored libs via **git submodules** (clone with `--recursive`, or run
`git submodule update --init --recursive`).

Use the CMake presets in `CMakePresets.json`:

```bash
# Configure (pick the preset for your platform)
cmake --preset debug-linux      # or debug-windows / debug-macos-default / debug-macos-xcode
# Build
cmake --build build/debug -j
```

Manual configure (matches CI), if not using presets:

```bash
cmake -S . -B build/ -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
  -DCMAKE_BUILD_TYPE=Debug -DWITH_TESTS=ON -DWITH_VST3=ON
cmake --build build/ -j 3
```

Key CMake options: `WITH_TESTS` (compiles the Catch2 test suite into the binary),
`WITH_VST3` (VST3 plugin support).

### Tests

There is **no separate test binary**. With `WITH_TESTS=ON`, the test files are `#include`d directly
into `src/core/init.cpp` and compiled into the `giada` executable. Run them by passing
`--run-tests`, which hands control to a Catch2 session:

```bash
./build/giada --run-tests              # all tests (Linux needs xvfb-run, it links GUI)
./build/giada --run-tests "[wave]"     # filter by Catch2 tag
./build/giada --run-tests "Wave*"      # filter by test-case name
```

Arguments after `--run-tests` are forwarded straight to Catch2, so any Catch2 CLI selector works.
On Linux the binary still links FLTK, so wrap test runs in `xvfb-run`. Test sources live in
`tests/`; **a new test file must also be added to the `#include` list in `src/core/init.cpp`** to
be compiled and run. Test fixtures/sample files are in `tests/resources/` and mocks in
`tests/mocks/`.

Formatting is enforced via `.clang-format` (Microsoft base, Allman braces, tabs for indentation,
no column limit).

## Architecture

The codebase is split into three layers, each its own namespace, communicating in one direction:

- **`src/core/`** — `giada::m` (model). The audio engine and all non-UI logic. UI-independent.
- **`src/glue/`** — `giada::c` (controller). The bridge between GUI and core; every user action
  routes through a glue function. When wiring a GUI control to engine behavior, add/extend a glue
  function rather than calling core directly from a widget.
- **`src/gui/`** — `giada::v` (view). FLTK widgets and windows. `src/gui/elems/` holds reusable
  widgets, `src/gui/dialogs/` the windows.

Entry point is `src/main.cpp`: it constructs a single `m::Engine` and a single `v::Ui`, exposed as
the globals `g_engine` / `g_ui`, then calls `m::init::startup()` / `m::init::run()`.

### Engine (`src/core/engine.h`)

`m::Engine` owns every core subsystem (KernelAudio, KernelMidi, Mixer, Sequencer, PluginHost,
Recorder, the various Managers, etc.) and the central `model::Model`. External code does **not** poke
subsystems directly — it goes through the **Api objects** the Engine exposes
(`MainApi`, `ChannelsApi`, `PluginsApi`, `SampleEditorApi`, `ActionEditorApi`, `IOApi`, `StorageApi`,
`ConfigApi`, in `src/core/api/`). The glue layer calls these Apis.

### The model and realtime threading (most important to understand)

State lives in `model::Model` (`src/core/model/`) as a single `Document`. The realtime audio thread
and the main thread share it through a **double-buffered atomic swap** (`mcl-atomic-swapper`), not
locks held across the audio callback.

The pattern for mutating shared state from a non-realtime thread:

1. Take a scoped lock: `model.lockShared()` (returns a `SharedLock`), or get a mutable reference via
   `model.get()`.
2. Mutate the `Document`.
3. `model.swap(SwapType)` publishes the change; the realtime thread picks up the new copy atomically.

The realtime thread reads through `model.get_RT()` (a `DocumentLock` giving read-only access). Code
running in the audio callback must stay realtime-safe: no allocation, locking, or I/O. Larger objects
(Waves, Plugins, ChannelShared) are stored in a shared pool and referenced by `ID`; add/find/remove
them via the `Model` methods (`addWave`, `findPlugin`, `removeChannelShared`, …). `onSwap` /
`Engine::onModelSwap` callbacks let the UI react to model changes.

### Rendering & reactions (`src/core/rendering/`)

The audio-callback render path is isolated here. `Renderer` produces audio; `Reactor` handles
events that mutate playback. Files are split by domain into `sample*` and `midi*` (advance, reactions,
output) plus `pluginRendering`. Channel types live in `src/core/channels/` (`sampleChannel`,
`midiChannel`, with the shared/atomic part in `channelShared`).

### Persistence

Projects and config serialize to JSON (nlohmann-json). `Patch` = a saved project, `Conf` = app
configuration; each has a matching `*Factory` for (de)serialization. `StorageApi` / `glue/storage.cpp`
drive load/save.

### Plugins

VST3 hosting is built on JUCE (`src/deps/juce`) and the Steinberg VST3 SDK (`src/deps/vst3sdk`).
`PluginHost`, `PluginManager`, and `src/core/plugins/` manage scanning, instantiation, and the
realtime plugin processing chain.

## Conventions

- Namespaces map to layers: `giada::m` (core), `giada::c` (glue), `giada::v` (gui); sub-namespaces
  like `giada::m::model`, `giada::m::rendering`, `giada::c::main` follow the directory.
- Include paths are repo-root-relative (e.g. `#include "src/core/engine.h"`), not relative to the
  current file.
- Header guards use the `G_` prefix (e.g. `G_ENGINE_H`).
- Debug-only code is gated behind `G_DEBUG_MODE`; JACK-specific code behind `WITH_AUDIO_JACK`.
- Keep core (`giada::m`) free of GUI dependencies — it must be testable and UI-independent.
