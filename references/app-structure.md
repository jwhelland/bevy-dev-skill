# App Structure: Plugins, Schedules, States, Ordering

Targets Bevy 0.18.

## App and plugins

Everything is registered on `App`; group registrations into plugins. Every
nontrivial feature should be a plugin (see `patterns.md` for organization).

```rust
pub struct CombatPlugin;

impl Plugin for CombatPlugin {
    fn build(&self, app: &mut App) {
        app
            .init_resource::<CombatRules>()
            .add_message::<DamageDealt>()
            .add_systems(Update, (apply_damage, check_death).chain())
            .add_observer(on_unit_died);
    }
}

App::new()
    .add_plugins(DefaultPlugins)
    .add_plugins((CombatPlugin, UiPlugin, AudioPlugin))
    .run();
```

- `DefaultPlugins` = windowing, rendering, input, audio, etc. Configure
  members with `.set(...)`:

```rust
DefaultPlugins
    .set(WindowPlugin {
        primary_window: Some(Window { title: "Game".into(), ..default() }),
        ..default()
    })
    .set(ImagePlugin::default_nearest())   // pixel art
```

- `MinimalPlugins` for headless (servers, tests). A headless app loops at
  max speed; add `ScheduleRunnerPlugin::run_loop(Duration::from_secs_f64(1.0/60.0))`
  to throttle.
- Plugins can be `fn(&mut App)` functions too — fine for small ones.
- Parameterize plugins with struct fields, not global state.

## Schedules

Main schedules in execution order each frame:

`First` → `PreUpdate` → `StateTransition` → `RunFixedMainLoop` (runs
`FixedUpdate` 0..n times) → `Update` → `PostUpdate` → `Last`.

`Startup` (with `PreStartup`/`PostStartup`) runs once before the first
frame. Rendering runs in a separate extracted schedule — user code rarely
touches it.

Rules of thumb:

- Game logic → `Update`.
- Physics/deterministic simulation → `FixedUpdate` (see below).
- Setup → `Startup` or `OnEnter(state)`.
- Reacting to engine-internal propagation (transform/visibility) →
  `PostUpdate` ordered against the engine sets (`TransformSystems::Propagate`,
  set names use the `*Systems` suffix since 0.17).

## System ordering

Within a schedule, order is nondeterministic unless constrained:

```rust
app.add_systems(Update, (
    input_handling,
    (movement, collision).chain(),          // chain = run in this order
    camera_follow.after(movement),
    hud_update.run_if(in_state(GameState::Playing)),
));
```

`.before()` / `.after()` / `.chain()` create ordering edges. Ordering does
not prevent parallelism for non-conflicting systems; it only constrains
when they may start.

### System sets

Named groups for cross-plugin ordering:

```rust
#[derive(SystemSet, Debug, Clone, PartialEq, Eq, Hash)]
enum GameSet { Input, Logic, Effects }

app.configure_sets(Update, (GameSet::Input, GameSet::Logic, GameSet::Effects).chain());
app.add_systems(Update, movement.in_set(GameSet::Logic));
```

Configure set relationships once (usually in a top-level plugin); plugins
then place systems into sets without knowing about each other.

### Ambiguity detection

Conflicting, unordered system pairs are a latent bug source. Enable
reporting during development:

```rust
app.edit_schedule(Update, |s| {
    s.set_build_settings(ScheduleBuildSettings {
        ambiguity_detection: LogLevel::Warn, ..default()
    });
});
```

## Run conditions

```rust
app.add_systems(Update, (
    pause_menu.run_if(in_state(GameState::Paused)),
    autosave.run_if(on_timer(Duration::from_secs(30))),
    sync.run_if(resource_exists::<Server>.and(resource_changed::<WorldState>)),
    cheat.run_if(input_just_pressed(KeyCode::F12)),
));
```

Combinators: `.and()`, `.or()`, `not(...)`. A run condition is any
read-only system returning `bool`. Conditions on a tuple of systems are
evaluated once for the whole tuple; conditions on a set gate the set.

## States

```rust
#[derive(States, Debug, Clone, PartialEq, Eq, Hash, Default)]
enum GameState { #[default] Menu, Playing, Paused }

app.init_state::<GameState>();

app.add_systems(OnEnter(GameState::Playing), spawn_level);
app.add_systems(OnExit(GameState::Playing), teardown_level);
app.add_systems(Update, gameplay.run_if(in_state(GameState::Playing)));

fn start(mut next: ResMut<NextState<GameState>>) {
    next.set(GameState::Playing);
}
```

- Transitions happen in the `StateTransition` schedule (after `PreUpdate`).
  `OnEnter`/`OnExit`/`OnTransition` schedules run then.
- **0.18 change**: `next.set(same_state)` *re-triggers* the transition
  (OnExit+OnEnter run again). Use `set_if_neq` for the old "only if
  different" behavior.
- **SubStates**: state that only exists within a parent state.

```rust
#[derive(SubStates, Debug, Clone, PartialEq, Eq, Hash, Default)]
#[source(GameState = GameState::Playing)]
enum PausedState { #[default] Running, Paused }
app.add_sub_state::<PausedState>();
```

- **ComputedStates**: derived read-only states (implement `ComputedStates`
  with a `compute` fn over source states).
- **State-scoped entities**: spawn with `DespawnOnExit(GameState::Playing)`
  to auto-despawn on leaving the state — usually better than manual
  teardown systems. (`DespawnOnEnter` also exists.)

## Fixed timestep

`FixedUpdate` runs at `Time<Fixed>` rate (default 64 Hz) — zero or more
times per frame to catch up.

```rust
app.insert_resource(Time::<Fixed>::from_hz(60.0));
app.add_systems(FixedUpdate, physics_step);
```

- Inside `FixedUpdate`, `Res<Time>` automatically yields fixed delta.
- Put simulation (physics, AI ticks) in `FixedUpdate`; put rendering-coupled
  logic (camera, animation, input *sampling*) in `Update`.
- Input gotcha: `just_pressed` can be missed or double-seen in
  `FixedUpdate`. Read input in `Update` and forward via messages/components,
  or use `RunFixedMainLoopSystems::BeforeFixedMainLoop`.
- Visual smoothness at low tick rates needs interpolation of transforms
  (do it yourself or via physics-crate options like avian's interpolation).

## Sub-apps and the render world

Rendering extracts data into a separate world each frame. You only care
when writing custom render features — see `rendering-3d.md`. Don't store
gameplay state in the render world.
