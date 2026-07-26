# CLAUDE.md — Custom C++ Game Engine

---

## 1. What this project is

A cross-platform game engine built from scratch in modern C++, for a developer who is an experienced programmer but new to engine development. **The primary goal is deep understanding, not feature count.** Favor clean architecture, correct dependency structure, and inspectable abstractions over speed of feature delivery.

This is an educational project. When a decision touches hardware/OS behavior (GPU/CPU synchronization, threading, cache layout, context/thread affinity), leave a **short explanatory comment** so the human learns the *why* from the code. Comment **why, not what**.

**Structural axiom:** the engine is a **library**; games are **separate executables that link against it**. If the test game (`Sandbox`) ever needs to include an *internal* engine header to function, that's a hole in the public API — flag it, don't paper over it.

---

## 2. Target platform & toolchain (locked)

| Decision | Value | Reason |
|---|---|---|
| Primary OS | **Windows first** | Dev machine. Linux/macOS support arrives with the future Vulkan backend. |
| Compiler / IDE | **MSVC / Visual Studio 2022** | Human's environment. |
| Language | **C++20** (no modules) | Concepts, ranges, `std::span`, `std::format`, `<=>` are all useful; C++20 *modules* fight the toolchain — avoid for now. |
| Graphics API | **OpenGL 4.6 core profile** | Latest GL; gives DSA + debug callback. Isolated behind the renderer for an eventual Vulkan port. |
| Build system | **CMake** (target-based, modern) | Industry standard; every dependency ships CMake support. |
| Dependency fetch | **FetchContent** initially | Reproducible, no submodule/package-manager ceremony. Graduate to vcpkg only if dep count grows. |
| Engine artifact | **Static library** | Avoids C++ DLL-boundary hazards (ABI, template export, allocator mismatch). Hot-reload DLLs are a targeted later feature, not a default. |

### Build configurations
- **Debug** — asserts on, logging on, no optimization, GL debug callback synchronous.
- **Release** — optimized, asserts on, logging on.
- **Dist** — shipping build: asserts compiled out, logging stripped to warnings/errors or off, GL debug callback disabled.

### GPU note (NVIDIA RTX 3070)
NVIDIA's driver is **permissive**: its GLSL compiler accepts non-conformant code and it tolerates state mistakes that AMD/Intel reject. **Do not rely on this leniency.** To stay portable:
- Always request a **core** profile context (no compatibility fallback).
- Shaders: pin `#version 460 core`, use **explicit `layout(location=...)`** on all vertex attributes and I/O, no implicit type conversions.
- Enable `glDebugMessageCallback` (core since 4.3) in Debug/Release, routed into our logger, `GL_DEBUG_OUTPUT_SYNCHRONOUS` on so callbacks fire on the offending call.
- Prefer **Direct State Access** (`glNamed*`, core since 4.5) over bind-to-edit. It avoids stale-binding bugs and mirrors how Vulkan/D3D12 think in explicit objects — easing the future port.

---

## 3. Architecture: layered, one-directional dependencies

Lower layers **must not** know about higher layers. Dependencies point **down**; data flows **up**.

```
Tools / Editor (ImGui-based, later)
Scene / ECS / Gameplay
Renderer (high-level: materials, meshes, render graph)
RHI (thin wrap over OpenGL; a real abstraction comes LATER, not now)
Resources / Assets
Platform (windowing, input, files)
Core (log, assert, events, memory, math, containers, time)
```

**Enforce layering via CMake target visibility** (`PRIVATE` vs `PUBLIC` link/include) wherever practical — the build should resist illegal dependencies, not just code review.

**Do not build a renderer-agnostic RHI abstraction yet.** Write straightforward OpenGL wrapper classes first (`Shader`, `Texture2D`, `VertexBuffer`, `VertexArray`), feel the seams, and refactor toward a true RHI once Vulkan's constraints are understood. This is *deliberate, planned* technical debt — note it in the decision log, don't pre-abstract.

### The mesh split (bakes the dependency rule into a concrete rule)
An asset is **pure data** and must be loadable **without a GPU** (needed for offline asset cooking, a headless server, and unit tests). Therefore:
- **`MeshAsset`** — CPU-side only: vertex bytes, index array, a `VertexLayout`, bounding box. Lives in **Resources**. No GL calls, no GL context required.
- **`GpuMesh`** — GL object handles (VBO/IBO/VAO). Lives in **Renderer**. The renderer maps asset → GPU resource and owns its lifetime.
- **`VertexLayout`** and other shared graphics enums/PODs are **pure data with no behavior and no dependencies** — put them in a small graphics-types header near the **bottom** of the stack. Assets produce it; the renderer consumes it. Assets are **self-describing**; the renderer must never assume a fixed vertex format.

Litmus test for any asset type: *can this be loaded and manipulated with no OpenGL context alive?* If not, it's mis-layered.

---

## 4. Third-party dependencies

**Vendored / FetchContent (pin versions):**
- **GLFW** — windowing, GL context, input. Must appear **only** inside `Platform/GLFW/`, never in a public header.
- **GLAD** — GL function loader, generated for **GL 4.6 core**. Build-time.
- **GLM** — math (GLSL-mirroring, column-major, matches GL). Header-only.
- **stb_image** — image decode. Becomes a tool-side dep once a real asset pipeline exists.
- **Dear ImGui** — debug UI, integrated ~Phase 2. Never a public API dependency of the engine.
- **spdlog** — logging backend, wrapped behind our own `Log` facade (below).
- **doctest** — unit tests (fast compiles).

**Deliberately deferred — do NOT add without asking:** Assimp (hand-parse OBJ first), an audio lib (miniaudio, later), physics (Jolt, much later), and **EnTT** — the ECS is being **built from scratch** in Phase 4 as an educational exercise, with EnTT studied only as a reference implementation.

**General rule:** buy commodity plumbing, hand-build the architecturally instructive parts. Do not introduce any new dependency silently — propose it, state the tradeoff, wait for confirmation.

---

## 5. Directory structure

```
Engine/                     # the engine (static library target)
├── CMakeLists.txt
├── src/
│   ├── Core/               # Log, Assert, Events/, Timestep, UUID
│   ├── Platform/
│   │   ├── Window.h        # abstract interface
│   │   └── GLFW/           # concrete impl — hidden from public API
│   ├── Renderer/
│   │   └── OpenGL/         # concrete GL objects — hidden behind Renderer/
│   ├── Scene/
│   └── Resources/
Sandbox/                    # test game (executable) that links Engine
├── CMakeLists.txt
└── src/
vendor/                     # third-party
assets/                     # shaders, textures, test models
tests/                      # doctest suites
CMakeLists.txt              # top-level workspace
```

---

## 6. Coding conventions

- **Namespace:** everything under `Engine` (rename to real engine name). Nested namespaces sparingly.
- **Types** (class/struct/enum class): `PascalCase`.
- **Functions & methods:** `PascalCase` (engine-style public API).
- **Private members:** `m_MemberName`. **Statics:** `s_Name`. Avoid globals; if unavoidable, `g_Name`.
- **Locals / parameters:** `camelCase`.
- **`enum class` values:** `PascalCase`. **Macros:** `ENGINE_` prefix, `SCREAMING_SNAKE`.
- **Headers:** `#pragma once`. A **precompiled header** carries the heavy std/vendor includes.
- **Memory & ownership:** RAII everywhere; Rule of Zero by default, Rule of Five when managing a resource. Make ownership explicit (`unique_ptr` for sole ownership, `shared_ptr` only when sharing is real). Be conscious that node-based/pointer-chasing structures cost cache misses — prefer contiguous storage in hot paths and say so in a comment when it matters.
- **Errors:** `ENGINE_ASSERT(cond, msg)` / `ENGINE_CORE_ASSERT(cond, msg)` for programmer errors (invariants), compiled out in Dist. Reserve exceptions for exceptional, non-hot-path failures; never for control flow in the frame loop.
- Explain any **non-obvious language feature** (concepts, `if constexpr`, fold expressions, custom deleters) in a brief comment.

---

## 7. Logging & assert facade (Phase 0 deliverable)

- Wrap spdlog behind our own `Log`. Two channels: **core** (engine) and **client** (game): `ENGINE_CORE_TRACE/INFO/WARN/ERROR/CRITICAL` and `ENGINE_TRACE/INFO/WARN/ERROR/CRITICAL`.
- `Log::Init()` sets pattern (timestamp, logger name, level) and levels per build config. Macros compile down/out in Dist.
- Assert macros use `__debugbreak()` on MSVC, log file/line/message, and vanish in Dist.

---

## 8. Entry-point inversion (Phase 0 deliverable)

Invert control so the **engine owns `main()`**:
- Engine provides the `main()` (via an `EntryPoint.h` included exactly once by the client, or compiled into the lib against a declared factory).
- The game **implements** `Engine::Application* Engine::CreateApplication()`.
- Engine's `main()` calls `Log::Init()`, constructs the app via the factory, runs it, and deletes it — with clean startup/shutdown ordering.
- This keeps platform/bootstrap details inside the engine and gives every game a uniform, minimal entry surface. Leave a comment explaining the IoC rationale.

---

## 9. CURRENT SCOPE — Phase 0 only

**Do not run ahead into windowing, rendering, or ECS.** Phase 0 is plumbing:

1. Top-level CMake workspace; `Engine` (static lib) + `Sandbox` (exe) with correct target-based linking and include visibility.
2. Precompiled header for the engine.
3. `Log` facade over spdlog (core + client channels, per-config levels).
4. Assert macros with debug-break, compiled out in Dist.
5. Entry-point inversion: engine `main()` + `CreateApplication()` factory; `Sandbox` supplies a trivial `Application` subclass.
6. Build configs: Debug / Release / Dist.

**Deliverable:** a window-less `Sandbox` that initializes logging, prints a startup line through both channels, runs an empty app, and shuts down cleanly. Builds in all three configs under MSVC.

Keep everything portable-friendly (no gratuitous Windows-only code outside a `Platform/Windows/` area) even though only Windows is built now.

---

## 10. Roadmap (for trajectory awareness — do not implement ahead)

- **P1** Platform & events: abstract `Window` + GLFW impl, GL 4.6 context, `glDebugMessageCallback`, event system, layer stack.
- **P2** Renderer foundations: `Shader`, buffers, `VertexArray` w/ layout API, textures, cameras (ortho→perspective), ImGui integration.
- **P3** Renderer architecture: submit interface, materials, 2D batch renderer, framebuffers, on-screen stats.
- **P4** Scene & ECS **from scratch** (sparse sets, entity IDs w/ generation counters, component pools), transforms, serialization.
- **P5** Assets: handle-based manager, hot-reload shaders, mesh loading, import pipeline beginnings.
- **P6+** 3D lighting (Blinn-Phong → PBR), RHI refactor, profiling, jobs/threading, editor.

---

## 11. Working agreement for Claude Code

- **Stay within the current phase.** Don't pre-build later systems or pre-abstract the RHI.
- **No new dependencies without proposing them first** (state the tradeoff).
- **Respect the layering.** Never let a lower layer depend on a higher one; keep GLFW/GLAD/ImGui out of public headers.
- **Maintain an Architecture Decision Record** (`docs/decisions.md`): for each significant choice, record the decision, the alternatives, and the reason. Append to it as you go.
- **Comment the *why*,** especially for hardware/OS-level reasoning (this is a learning project).
- When you deviate from this spec or hit an ambiguous fork, **flag it and explain** rather than silently choosing.
- Prefer **small, reviewable increments** with clear commit messages over large drops.
