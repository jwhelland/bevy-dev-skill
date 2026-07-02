# Architecture & Best Practices

Targets Bevy 0.19. Read this before building any nontrivial feature.

## Project organization

Organize by *feature*, not by ECS kind. Avoid `components.rs` /
`systems.rs` dumping grounds.

```
src/
├── main.rs          # App assembly only: plugins, top-level sets, states
├── player/
│   ├── mod.rs       # PlayerPlugin; pub API of the feature
│   ├── movement.rs
│   └── abilities.rs
├── enemies/
├── level/
├── ui/
└── shared/          # cross-feature components/messages (keep small)
```

Each feature = one plugin owning its components, systems, messages,
events, and assets. `main.rs` stays a table of contents. Inter-feature
communication goes through messages/events/shared components — never by
reaching into another plugin's systems.

Cross-cutting order: define global `SystemSet`s in a core plugin
(`GameSet::{Input, Logic, Effects}` pattern, `app-structure.md`), configure
them once, and have feature plugins place systems into sets. Plugins stay
order-independent at registration time.

## Component design

- **Small and single-purpose.** `Position`, `Velocity`, `Health` — not
  `PlayerData { everything }`. Parallelism, change detection, and reuse all
  improve with granularity.
- **Marker components are free.** `Player`, `Enemy`, `MainCamera`,
  `Selected` — zero-sized types used in filters. Use them liberally instead
  of enum-typed "kind" fields you'd have to branch on (`With<Enemy>` beats
  `if kind == Kind::Enemy` — the archetype does the filtering).
- **Newtype wrappers** give meaning and prevent mix-ups:
  `struct MoveSpeed(f32)`, `struct Damage(f32)`.
- **Composition over inheritance.** A "flying poisonous boss" =
  `(Enemy, Flying, Poisonous, Boss)`. Behavior systems query the capability
  they implement (`Query<&mut Transform, With<Flying>>`).
- **Encode state as component presence**, not boolean fields, when systems
  differ by state: `Stunned` as an insertable marker beats `stunned: bool`
  (systems filter archetypally; add/remove triggers observers/hooks).
  Exception: very high-frequency toggles — archetype moves have a cost.
- Use `#[require(...)]` to make invalid combinations unrepresentable
  (`ecs-core.md`).

## System design

- One job per system; name = what it does (`apply_velocity`,
  `despawn_dead`). Many small systems parallelize and read better.
- Narrow queries: ask only for the components you use; prefer `&T` over
  `&mut T` whenever possible (read-only systems parallelize freely).
- Use run conditions and states to avoid "if not active, return" bodies.
- Don't over-order: add `.before`/`.after`/`.chain` only where a real data
  dependency exists; let the scheduler do the rest. Turn on ambiguity
  detection in dev (`app-structure.md`).
- Generic systems (`fn cleanup<T: Component>(...)`) for repeated patterns:

```rust
fn despawn_all<T: Component>(mut commands: Commands, q: Query<Entity, With<T>>) {
    for e in &q { commands.entity(e).despawn(); }
}
app.add_systems(OnExit(GameState::Playing), despawn_all::<LevelEntity>);
// (For state teardown specifically, prefer DespawnOnExit — app-structure.md)
```

## Communication decision table

| Need | Use |
|---|---|
| Stream of facts processed in bulk per frame | `Message` |
| "Something happened to entity X" reaction | `EntityEvent` + observer |
| Component invariant enforcement | hook |
| Share state across systems continuously | component or resource |
| Call a named piece of logic on demand | one-shot system |
| Cross-frame flags / modes | `States` or resource |

Antipattern: chains of observers triggering observers for core game flow —
hidden control flow, unspecified ordering. Main loop logic should be
visible in the schedule.

## Common architectural patterns

**Intent/actuation split.** Input systems write intent components
(`MoveIntent(Vec2)`, `WantsToJump`); simulation systems consume intent.
Decouples devices from logic, makes AI and replay trivial (AI writes the
same intent components).

**Cooldowns/durations**: `Timer` in a component (`transforms-time.md`).

**Delayed despawn**: `struct Lifetime(Timer)` + one system ticking and
despawning. Reuse everywhere (particles, projectiles, popups).

**Object pooling**: usually unnecessary — spawn/despawn is fast. If proven
hot, use `Disabled` (`ecs-advanced.md`) to park entities.

**Singleton entity** (player, camera): marker + `Single<...>` param. Don't
cache `Entity` ids in resources unless needed across frames.

**Level/screen lifecycle**: state + `DespawnOnExit(state)` on everything
spawned for that screen. Never hand-track lists of spawned entities.

**Asset preloading**: a `Loading` state and a handle-holding resource
(`assets.md`).

**Fixed-timestep simulation** with visual interpolation for anything
physics-like (`app-structure.md`).

## Error handling

Systems can return `Result`; use `?` instead of `unwrap()` on queries and
asset lookups. Default handler panics in dev; set a warn-handler for
release builds (`App::set_error_handler(bevy::ecs::error::warn)`).
`Single`/`Populated` params encode "skip when absent" without any error
handling at all. Avoid `unwrap()` in systems — entity/asset existence is
rarely guaranteed across frames.

## Performance habits (in order of payoff)

1. Build with `opt-level = 1` dev profile + `dynamic_linking` for iteration
   speed; **measure release builds** for actual perf.
2. Don't do per-frame work that can be event/change-driven
   (`Changed<T>` filters, observers).
3. Share mesh/material handles for repeated objects (auto-batching).
4. Keep `Assets::get_mut` out of hot loops (GPU re-upload).
5. Use `FixedUpdate` for simulation to bound work.
6. `par_iter_mut().for_each(...)` on big queries only after profiling shows
   the system is hot, the per-entity work is real, and it doesn't use Commands.
7. Profile with Tracy (`trace_tracy` feature) before optimizing anything.

## Things people port wrongly from OOP engines

- No "GameObject with methods": entities have no behavior; don't write
  `impl Player { fn update(&mut self) }`.
- No deep inheritance trees: flatten into marker + capability components.
- No global mutable singletons accessed from anywhere: resources are
  explicit system parameters, and that's the point — access is declared.
- Don't fight the scheduler with one mega-system "so order is clear";
  declare ordering between small systems instead.
- Don't store `&mut World` or entity references inside components; store
  `Entity` ids and look up via queries.
