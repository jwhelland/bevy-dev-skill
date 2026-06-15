# Testing and Debugging

Targets Bevy 0.18.

## Unit-testing systems

`bevy_ecs` works headless — no window, GPU, or asset server needed for
logic tests.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn damage_kills_at_zero() {
        let mut app = App::new();
        app.add_plugins(MinimalPlugins)
            .add_message::<DamageDealt>()
            .add_systems(Update, (apply_damage, despawn_dead).chain());

        let e = app.world_mut().spawn(Health { current: 5.0, max: 10.0 }).id();
        app.world_mut().write_message(DamageDealt { target: e, amount: 9.0 });

        app.update();   // run one frame
        app.update();   // commands from frame 1 apply; despawn visible now

        assert!(app.world().get_entity(e).is_err());
    }
}
```

Key tools:

- `app.update()` runs one frame of all schedules. Remember commands are
  deferred — effects of frame N's commands are queryable in frame N+1
  (or after the next sync point).
- `app.world_mut()` for direct setup/assertions:
  `world.spawn`, `world.get::<T>(e)`, `world.resource::<T>()`,
  `world.run_system_once(my_system)` (import
  `bevy::ecs::system::RunSystemOnce`) to test one system in isolation.
- Test a single plugin by adding just it + `MinimalPlugins`.
- Time-dependent tests: insert `Time` manually or tick with
  `app.world_mut().resource_mut::<Time<Virtual>>().advance_by(d)`.
- For input simulation: `world.resource_mut::<ButtonInput<KeyCode>>().press(KeyCode::Space)`
  (call `.clear()` between frames to reset just_pressed).

## Headless / CI

- `MinimalPlugins` (+ `ScheduleRunnerPlugin`) for logic-only apps/servers.
- Rendering code in CI without a display: run with
  `WgpuSettings { backends: None, .. }` via `RenderPlugin`, or use a
  software adapter (lavapipe/llvmpipe) if pixels matter.
- Asset-dependent tests: `DefaultPlugins` minus window
  (`.disable::<WinitPlugin>()`) or construct `AssetPlugin` with a test
  asset root.

## Logging & diagnostics

```rust
// Log level via env: RUST_LOG=info,wgpu=error,my_game=debug cargo run
// or LogPlugin { filter: "info,wgpu_core=warn".into(), .. }
info!("spawned {} enemies", n);  debug!(?entity, "tagged");  warn_once!("...");
```

Built-in diagnostics:

```rust
app.add_plugins((
    bevy::diagnostic::FrameTimeDiagnosticsPlugin::default(),  // FPS/frametime
    bevy::diagnostic::EntityCountDiagnosticsPlugin,
    bevy::diagnostic::LogDiagnosticsPlugin::default(),        // prints them
));
```

On-screen FPS overlay: `bevy::dev_tools::fps_overlay::FpsOverlayPlugin`
(feature `bevy_dev_tools`). UI layout debug overlay: `UiDebugOptions`
(`ui.md`). Visual debugging: gizmos (`rendering-3d.md`).

Entity inspection at runtime: `bevy-inspector-egui` (ecosystem) is the
de-facto inspector — list/edit entities, components, resources live.

## Profiling

```bash
cargo run --release --features bevy/trace_tracy   # connect Tracy profiler ≥0.11
```

Spans per system come free. Custom spans: `let _s = info_span!("ai").entered();`.
`spawn_timed` alternatives: `trace_chrome` feature writes a chrome://tracing
json. **Never profile debug builds.**

## Common runtime errors (B-codes)

Bevy errors carry codes; full explanations live at
`bevy_ecs::error` docs / bevy.org/learn/errors (offline: rustdoc).

- **B0001** — query/param aliasing: two params in one system mutably access
  the same component. Fix: `Without<>` filters or `ParamSet`
  (`ecs-core.md`).
- **B0002** — `ResMut<T>` + `Res<T>` (or two `ResMut<T>`) in one system.
- **B0003** — command applied to a despawned entity (double-despawn,
  insert-after-despawn). Find with the log's entity id; dedupe despawn
  responsibility, use `try_despawn`/`try_insert` for benign races.
- **B0004** — child entity missing a component its parent's hierarchy needs
  (e.g. spawned UI child without `Node`... ): add the required component
  or spawn via `children![]` so requires propagate.

## Common panics & their causes

| Symptom | Likely cause |
|---|---|
| `Resource requested by system does not exist` | forgot `init_resource`/`insert_resource`, or plugin order (use `Option<Res<T>>` if legitimately optional) |
| Query `single()` Err / `Single` skipping | zero or >1 matches; spawn order race on frame 1 — gate with `run_if` or use `Option<Single>` |
| Message never received | forgot `add_message`, or reader gated off >2 frames (events dropped), or writer runs after reader (1-frame lag) |
| Schedule build panic: cyclic dependency | contradictory `.before/.after` constraints |
| `Attempted to access entity ... that does not exist` | stale `Entity` id stored in component/resource; re-validate with `query.get` |
| Asset never loads | wrong path/case, missing feature for the format (e.g. `png`, `jpeg`), file outside `assets/` |
| Nothing renders | no camera; see per-domain gotchas in rendering-2d/3d.md |

## Debugging workflow tips

- Print archetype/component info: `world.inspect_entity(e)` yields
  component infos; `Name` components make logs readable
  (`Query<&Name>` + `?name` in tracing macros).
- `bevy mod picking`-style issues (clicks not registering): check
  `Pickable` presence and feature flags (`ui_picking`, `sprite_picking`,
  `mesh_picking`).
- Schedule introspection: `bevy_mod_debugdump` (ecosystem) renders
  schedule/system-order graphviz — invaluable for ordering bugs.
- Determinism issues usually trace to unordered systems racing (ambiguity
  detection) or HashMap iteration order — use `EntityHashMap`/sorted iteration.
