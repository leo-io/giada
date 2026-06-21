# AGENTS.md

Compact agent guide. `CLAUDE.md` has deeper architecture notes — read it for the
realtime/memory model; this file covers the things you'd get wrong by guessing.

## Setup & build

- Set `VCPKG_ROOT` before configuring (vcpkg manifest mode, `vcpkg.json`).
- Init submodules: `git submodule update --init --recursive` (deps under `src/deps/`).
- Use CMake presets, not raw `cmake -S . -B build`:
  - `cmake --preset debug-windows` (also: `debug-linux`, `debug-macos-default`, `debug-macos-xcode`)
  - `cmake --build build/debug -j`
  - Binary dirs are `build/debug` or `build/release` (per preset), not `build/`.
- Debug presets enable `WITH_TESTS=ON` and `WITH_VST3=ON` by default.
- CMakeLists requires **C++23** (`cxx_std_23`) — `CLAUDE.md` says C++20, but the
  build is C++23. Use C++23 features freely; don't downgrade.
- Windows build needs the ASIO SDK (pulled into RtAudio via `src/deps/rtaudio/include`).
- VST2 support requires `-DVST2_SDK_PATH=...` (off by default); VST3 is on by default.

## Tests (non-obvious — read this before touching tests)

- **There is no separate test binary.** Test files in `tests/` are `#include`d
  directly into `src/core/init.cpp:40-50` and compiled into the `giada` executable.
- When you add a new test file in `tests/`, you MUST also add its `#include` line to
  `src/core/init.cpp`'s include block, or it won't compile/run.
- Run: `./build/debug/giada --run-tests` — args after `--run-tests` forward to Catch2:
  - `--run-tests "[wave]"` (tag filter), `--run-tests "Wave*"` (name filter).
- On Linux the binary links FLTK, so wrap runs in `xvfb-run`:
  `xvfb-run ./build/debug/giada --run-tests`
- Test resources live in `tests/resources/` (path baked in via `TEST_RESOURCES_DIR`);
  mocks in `tests/mocks/`.
- Lint/typecheck: there is no lint step; formatting is enforced by `.clang-format`
  (run `clang-format -i <file>`). No separate typecheck target — the build is the check.

## Code style

- `.clang-format`: Microsoft base, **Allman braces**, **tabs** for indentation,
  **no column limit** (`ColumnLimit: 0`).
- Include paths are **repo-root-relative** (e.g. `#include "src/core/engine.h"`),
  never relative to the current file.
- Header guards use the `G_` prefix (e.g. `G_ENGINE_H`).
- Namespaces map to layers: `giada::m` (core), `giada::c` (glue), `giada::v` (gui);
  sub-namespaces mirror directories (`giada::m::model`, `giada::m::rendering`, …).
- Debug-only code: `#if G_DEBUG_MODE`; JACK-only code: `#ifdef WITH_AUDIO_JACK`.

## Architecture rules that change how you edit

Three layers, communication strictly one-directional for mutations:
`v (gui) → c (glue) → m (core)`. See `CLAUDE.md` for the full model.

- **Never call core directly from a GUI widget.** Route through a glue function in
  `src/glue/` (free functions in `giada::c::*`), which calls the appropriate
  `Engine::get<Name>Api()`. The 8 Apis are the only public surface of the engine.
- **The realtime audio thread (`Thread::AUDIO`) must stay RT-safe**: no allocation,
  no locks, no I/O, no `Document` mutation. Read state via `model.get_RT()` →
  `DocumentLock` (RAII); write back only through `a_*`-prefixed atomic methods or
  `ChannelShared` `WeakAtomic` fields / lock-free queues.
- **Non-RT mutation pattern**: `model.get()` (mutable ref) → mutate → `model.swap(SwapType)`
  to publish. `SwapType`: `HARD` (structure change), `SOFT` (property), `NONE` (no UI care).
- **Mutating the shared pool** (Waves/Plugins/ChannelShared) requires a `SharedLock`
  (sets `Document::locked=true`, so the renderer skips channel processing mid-mutation).
- **RT → main-thread work** goes through `EventDispatcher::pumpEvent(lambda)` (lock-free
  queue drained by a worker thread), never direct callbacks.
- **Keep core (`giada::m`) GUI-free** — it must be testable headless.
- Plugins (JUCE/VST3) must be instantiated and destroyed on the **main thread only**
  (JUCE/VST3 requirement).

## Entry points

- `src/main.cpp` — constructs one `m::Engine` and one `v::Ui` (globals `g_engine`/`g_ui`),
  then `m::init::startup()` / `m::init::run()`.
- `src/core/engine.h` — `Engine` owns all subsystems + `model::Model`; exposes 8 Apis.
- `src/core/rendering/renderer.cpp` — the audio-callback render path (`Renderer::render`).
- `src/core/rendering/reactor.cpp` — non-RT reactions to user actions (ends with `model.swap()`).
