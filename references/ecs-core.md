# ECS Core: Entities, Components, Systems, Queries

Targets Bevy 0.19. For hooks, observers, relationships, change detection →
`ecs-advanced.md`.

## Components

A component is any `'static` Rust type deriving `Component`. Keep them small
and data-only — behavior belongs in systems.

```rust
#[derive(Component)]
struct Health { current: f32, max: f32 }

#[derive(Component)]
struct Player;              // marker component: zero-sized, used for filtering

#[derive(Component, Default)]
#[require(Health::full(100.0), Transform)]   // required components (see below)
struct Enemy;
```

### Required components

`#[require(...)]` makes inserting one component automatically insert others
(if not already present). This replaced bundles as the idiomatic way to
guarantee invariants — e.g. `Camera3d` requires `Camera`, `Transform`, etc.
You can require with default values (`Transform`), expressions
(`Health::full(100.0)`), or closures. Requires apply recursively.

Bundles (`#[derive(Bundle)]` or tuples of components) still exist and are
fine for grouping spawn data, but prefer required components for "this
component never makes sense without that one".

## Spawning and despawning

```rust
fn spawn_things(mut commands: Commands, assets: Res<AssetServer>) {
    // Tuple of components = a bundle
    let id = commands.spawn((
        Enemy,
        Sprite::from_image(assets.load("enemy.png")),
        Transform::from_xyz(100.0, 0.0, 0.0),
    )).id();

    // With children (0.18: children! macro)
    commands.spawn((
        Player,
        Transform::default(),
        children![
            (Sprite::from_image(assets.load("hat.png")), Transform::from_xyz(0.0, 8.0, 1.0)),
        ],
    ));

    commands.entity(id).insert(Health { current: 10.0, max: 10.0 });
    commands.entity(id).remove::<Enemy>();
    commands.entity(id).despawn();   // also despawns children (relationship cleanup)
}
```

**Commands are deferred.** They apply at the next sync point (between
systems), not immediately. A system that spawns an entity cannot query for
it in the same system run. For immediate effect use exclusive systems
(`&mut World`) — see `ecs-advanced.md`.

## Systems

Any function whose parameters are all `SystemParam`s is a system. Common
params:

| Param | Access |
|---|---|
| `Query<D, F>` | entity/component data |
| `Res<T>` / `ResMut<T>` | resource (shared/exclusive) |
| `Option<Res<T>>` | resource that may not exist |
| `Commands` | deferred structural changes |
| `Local<T>` | per-system persistent state |
| `MessageReader<M>` / `MessageWriter<M>` | buffered messages |
| `Single<D, F>` | exactly one matching entity (system skips otherwise) |
| `Option<Single<D, F>>` | zero or one (None also when >1 in 0.17+) |
| `Populated<D, F>` | like Query but skips system when empty |
| `&mut World` | exclusive — no parallelism |

Bevy runs systems in parallel when their access doesn't conflict. Two
systems with `ResMut<T>` of the same `T`, or overlapping mutable queries,
serialize automatically. Smaller access = more parallelism.

Systems may return nothing or `Result` (`bevy::ecs::error::Result`). With a
`Result` return you can use `?` on fallible ops (`query.single()?`,
`assets.get(...).ok_or(...)?`); errors go to the configured error handler
(panic in dev by default — configurable via `App::set_error_handler`).

```rust
fn regen(mut q: Query<&mut Health>, time: Res<Time>) {
    for mut hp in &mut q {
        hp.current = (hp.current + 5.0 * time.delta_secs()).min(hp.max);
    }
}
```

## Queries

```rust
fn examples(
    q1: Query<&Transform>,                                    // read one component
    q2: Query<(&Transform, &mut Health), With<Enemy>>,        // filtered, mixed access
    q3: Query<(Entity, &Health), (With<Enemy>, Without<Boss>)>,
    q4: Query<&Health, Or<(With<Player>, With<Ally>)>>,
    q5: Query<Option<&Shield>, With<Player>>,                 // optional component
    q6: Query<EntityRef>,                                     // dynamic read access
) {}
```

- Filters: `With<T>`, `Without<T>`, `Or<(...)>`, `Added<T>`, `Changed<T>`
  (last two are change-detection filters — see `ecs-advanced.md`).
- Iterate: `for x in &query` (read) / `for mut x in &mut query` (write).
  `query.iter_combinations_mut::<2>()` for pairwise interactions.
- Single entity: `query.single()` / `single_mut()` return `Result` — use
  `?` in a Result-returning system, or prefer the `Single` param.
- By id: `query.get(entity)` / `get_mut(entity)` → `Result`.
- Multiple distinct entities mutably: `query.get_disjoint_mut([e1, e2])`
  (renamed from `get_many_mut` in 0.18).
- `query.contains(entity)`, `query.is_empty()`.

### The aliasing rule (most common beginner error, B0001)

Two queries in one system may not both mutably access the same component on
the same entities. Fix by making filters disjoint (`Without<...>`) or by
using a `ParamSet`:

```rust
// ERROR B0001: both queries can hit the same entity
// fn bad(a: Query<&mut Transform, With<Player>>, b: Query<&mut Transform>) {}

fn ok1(a: Query<&mut Transform, With<Player>>,
       b: Query<&mut Transform, Without<Player>>) {}

fn ok2(mut set: ParamSet<(Query<&mut Transform, With<Player>>, Query<&mut Transform>)>) {
    let _ = set.p0();   // only one active at a time
}
```

## Resources

```rust
#[derive(Resource, Default)]
struct Score(u32);

app.init_resource::<Score>();            // or .insert_resource(Score(0))

fn read(score: Res<Score>) { /* score.0 */ }
fn write(mut score: ResMut<Score>) { score.0 += 1; }
// In commands: commands.insert_resource(...) / remove_resource::<T>()
```

Use a resource when there is logically exactly one (settings, score, RNG
state). If "one per player/level" could ever apply, make it a component on
an entity instead.

As of 0.19, `Resource` is a subtrait of `Component` (resources are stored
as components on hidden singleton entities) — a type can no longer derive
both `Component` and `Resource`. Usually invisible day-to-day, but see
`ecs-advanced.md` for the query-conflict gotcha it introduces.

## Entity references

Store `Entity` ids inside components to reference other entities (e.g.
`struct Target(Entity)`). Always handle the target being despawned:
`query.get(target.0)` returns `Err` for dead entities. For 1:many
relationships with automatic cleanup, use the relationships feature
(`ecs-advanced.md`) instead of hand-rolled `Vec<Entity>`.

## Common pitfalls

- **Spawn-then-query in the same system**: deferred Commands; restructure
  or use chained systems.
- **`despawn` on an already-despawned entity** warns (B0003). Dedupe your
  despawn logic (one system owns despawning per entity type, or use
  `try_despawn`).
- **Forgetting `mut`** on the query binding: `for mut x in &mut q`.
- **Huge god-components**: split them; queries become cheaper and systems
  parallelize better.
- **`query.single()` panicking**: it returns `Result` now — handle it or
  use `Single`.
