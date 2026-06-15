---
name: bevy-dev
description: >-
  Expert guide for the Bevy game engine (Rust) with offline/airgapped-ready
  reference docs. ALWAYS use this skill — before writing any code or
  answering — when a task touches Bevy in any way: the user mentions Bevy,
  Cargo.toml depends on bevy, or the work involves Bevy ECS (entities,
  components, systems, queries, observers, messages, schedules, plugins),
  sprites, meshes, PBR materials, cameras, WGSL shaders, bevy_ui, assets,
  input, transforms, audio, animation, or scenes. Also use it for any Bevy
  compile error or panic (B0001-B0004), upgrading Bevy versions or fixing
  code written for an older Bevy, choosing Bevy ecosystem crates (avian,
  leafwing-input-manager, bevy_egui...), and architecting a Bevy project.
  Use it even for small "quick" Bevy snippets: Bevy APIs changed heavily
  across 0.15-0.18 and answers from memory often won't compile — this skill
  carries the current names and migration tables. If the user is making a
  Rust game with an ECS and the framework is unstated, use this skill.
---

# Bevy Development

This skill targets **Bevy 0.18** (released January 2026). Bevy's API changes
significantly between minor versions, so first check the project's actual
version (`grep 'bevy' Cargo.toml` or `cargo tree -p bevy --depth 0`). If the
project is on an older version, read [references/migration.md](references/migration.md) to translate —
it maps every major rename across 0.15 → 0.16 → 0.17 → 0.18 in both
directions.

## Ground truth: generated docs beat this skill

These reference files teach patterns, idioms, and architecture. For **exact
API signatures**, the generated rustdoc is authoritative and always matches
the project's pinned version:

```bash
cargo doc -p bevy --no-deps          # generate docs for the exact pinned bevy
cargo doc --open                      # docs for ALL dependencies, incl. third-party crates
```

This works fully offline. When unsure whether a method exists or what its
signature is, check the generated docs (or `rg` through
`~/.cargo/registry/src/`) rather than guessing. See
[references/offline-docs.md](references/offline-docs.md) for browsing/searching generated docs
efficiently on an airgapped machine.

## Core mental model (30 seconds)

- **World**: a database of entities. **Entity**: an opaque ID (a row).
  **Component**: plain data attached to an entity (a column). **Resource**:
  a global singleton. **System**: a normal Rust function whose parameters
  declare what data it reads/writes; Bevy schedules systems in parallel
  automatically based on those declarations.
- **App + Plugins**: an app is assembled from plugins; each plugin registers
  systems into **Schedules** (`Startup`, `Update`, `FixedUpdate`, ...).
- Two messaging styles: **Messages** (buffered, polled by systems via
  `MessageReader`/`MessageWriter`) and **Events + Observers** (push-style,
  run immediately when `trigger`ed, can target entities).
- **Commands** are deferred: structural changes (spawn/despawn/insert) queue
  up and apply at the next sync point, not instantly.
- Prefer many small components over few big ones. Logic lives in systems,
  not methods. Composition over inheritance, always.

## Which reference to read

Read only what the task needs — each file is self-contained.

| Task | Read |
|---|---|
| Queries, spawning, components, basic systems | [references/ecs-core.md](references/ecs-core.md) |
| Hooks, observers internals, change detection, EntityCommands, relationships, required components | [references/ecs-advanced.md](references/ecs-advanced.md) |
| App setup, plugins, schedules, states, system ordering, run conditions, fixed timestep | [references/app-structure.md](references/app-structure.md) |
| Messages vs events, observers, picking events | [references/events-observers.md](references/events-observers.md) |
| Loading assets, handles, custom loaders, hot reload, embedded assets | [references/assets.md](references/assets.md) |
| Sprites, 2D camera, atlases, Text2d, tilemap approach | [references/rendering-2d.md](references/rendering-2d.md) |
| Meshes, PBR, lights, 3D camera, shadows, gizmos, custom materials | [references/rendering-3d.md](references/rendering-3d.md) |
| UI nodes, layout, feathers widgets, scroll, picking on UI | [references/ui.md](references/ui.md) |
| Keyboard/mouse/gamepad/touch, window config, cursor | [references/input-windowing.md](references/input-windowing.md) |
| Transform, hierarchy, Time, Timers | [references/transforms-time.md](references/transforms-time.md) |
| Sound playback, animation graph | [references/audio-animation.md](references/audio-animation.md) |
| Reflect, scenes, serialization | [references/reflection-scenes.md](references/reflection-scenes.md) |
| Architecture & best practices for any nontrivial feature | [references/patterns.md](references/patterns.md) |
| Tests, headless apps, diagnosing panics & B-codes | [references/testing-debugging.md](references/testing-debugging.md) |
| "What crate should I use for physics/input/UI/...?" | [references/ecosystem.md](references/ecosystem.md) |
| Project is on Bevy < 0.18, or code uses old names | [references/migration.md](references/migration.md) |
| Airgapped setup, vendoring, local docs | [references/offline-docs.md](references/offline-docs.md) |

For any feature bigger than a single system, read
[references/patterns.md](references/patterns.md) too — it covers plugin organization, component
granularity, and the common architectural mistakes.

## Names that recently changed — write these, not the old ones

Tested models reliably slip on a handful of renames even when they know
0.18 broadly. Current names (left), common stale habit (right):

| Write this (0.18) | Not this |
|---|---|
| `timer.is_finished()` / `is_paused()` | `finished()` / `paused()` |
| `MessageReader` / `MessageWriter`, `writer.write(m)`, `app.add_message` | `EventReader`/`EventWriter`, `send`, `add_event` |
| `On<MyEvent>` observer param, `On<Add, T>` | `Trigger<MyEvent>`, `Trigger<OnAdd, T>` |
| `entity.despawn()` (recursive by default) | `despawn_recursive()` |
| `query.get_disjoint_mut([a, b])` | `get_many_mut` |
| `detach_children` / `detach_all_children` | `remove_children` / `clear_children` |
| `GlobalAmbientLight` resource | `AmbientLight` resource |
| `use bevy::ecs::lifecycle::HookContext` | `bevy::ecs::component::HookContext` |
| `use bevy::shader::ShaderRef` (`AsBindGroup` stays in `render::render_resource`) | `bevy::render::render_resource::ShaderRef` |
| `Gizmos::cube` | `Gizmos::cuboid` |
| `Justify`, `SystemCondition`, `*Systems` set names | `JustifyText`, `Condition`, `*Set`/`*System` |

Full mapping (and reverse, for older projects): [references/migration.md](references/migration.md).

## Project setup quick reference

```toml
[dependencies]
bevy = "0.18"
# Faster iterative compiles during development:
[profile.dev]
opt-level = 1
[profile.dev.package."*"]
opt-level = 3
```

Use `bevy = { version = "0.18", features = ["dynamic_linking"] }` only for
local dev iteration (never ship it). 0.18 added scenario feature collections:
`default-features = false, features = ["2d"]` (or `"3d"`, `"ui"`) to slim
builds. For fast linking, configure a faster linker in `.cargo/config.toml`
(lld on Linux/Windows; macOS default linker is already fast).

Minimal app:

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(Update, hello)
        .run();
}

fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}

fn hello(time: Res<Time>) {
    info!("t = {}", time.elapsed_secs());
}
```

## Version banner for generated code

When writing code for the user, state which Bevy version the code targets.
If the user's `Cargo.toml` pins an older version, write code for *their*
version using the migration tables — do not silently mix API generations
(e.g. `EventReader` from 0.16 with `children![]` from 0.18).
