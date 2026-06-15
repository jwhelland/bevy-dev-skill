# ECS Advanced: Change Detection, Hooks, Relationships, Exclusive Access

Targets Bevy 0.18. Prereq: `ecs-core.md`.

## Change detection

Every component and resource tracks `Added` and `Changed` ticks
automatically. Granularity is per-component-per-entity.

```rust
// As query filters (skip non-matching entities entirely):
fn on_hp_change(q: Query<(Entity, &Health), Changed<Health>>) {}
fn on_new_enemies(q: Query<Entity, Added<Enemy>>) {}

// As runtime checks (entity still iterated):
fn check(q: Query<Ref<Health>>) {
    for hp in &q {
        if hp.is_changed() && !hp.is_added() { /* changed since last run of THIS system */ }
    }
}

// Resources:
fn res_check(score: Res<Score>) { if score.is_changed() {} }
// Run condition versions: resource_changed::<Score>, resource_added::<Score>
```

Key facts:

- "Changed" means *dereferenced mutably*, not value-compared. Writing the
  same value still flags it. Use `Mut::set_if_neq(value)` or check before
  writing to avoid false positives.
- `Changed<T>` includes `Added<T>`.
- Detection is relative to the last time *this* system ran — systems that
  run rarely (run conditions, states) see accumulated changes; they don't
  miss any.
- Bypass: `mut_untyped`/`bypass_change_detection()` exists but is rarely
  worth it.
- `RemovedComponents<T>` is a system param (works like a message reader)
  for entities whose `T` was removed since last run — including despawns.

## Component hooks

Hooks are component-level lifecycle callbacks baked into the component
type — they *always* run, cannot be bypassed, and are good for enforcing
invariants (indexes, caches, ref-counting). For game logic prefer observers
(`events-observers.md`), which are more flexible.

```rust
use bevy::ecs::lifecycle::HookContext;   // note: lifecycle, not component
use bevy::ecs::world::DeferredWorld;

#[derive(Component)]
#[component(on_add = on_add_hook, on_remove = on_remove_hook)]
struct Tracked;

fn on_add_hook(mut world: DeferredWorld, ctx: HookContext) {
    // ctx.entity, ctx.component_id; DeferredWorld can read + queue commands
    world.commands().entity(ctx.entity).insert(Name::new("tracked"));
}
fn on_remove_hook(mut world: DeferredWorld, ctx: HookContext) {}
```

Hook points: `on_add`, `on_insert` (every insert incl. overwrite),
`on_replace` (before value replaced/removed), `on_remove`, `on_despawn`.
Order: hooks run before observers for add/insert; after observers for
replace/remove.

## Relationships

A relationship is a pair of components kept in sync by the engine:
the *relationship* (on the "child", points to one target) and the
*relationship target* (on the "parent", lists sources). `ChildOf`/`Children`
is the built-in pair.

```rust
#[derive(Component)]
#[relationship(relationship_target = Targets)]
struct Targeting(Entity);            // many-to-one side; single Entity field

#[derive(Component)]
#[relationship_target(relationship = Targeting, linked_spawn)]  // linked_spawn = despawn sources with target
struct Targets(Vec<Entity>);

// Usage: insert only the relationship side; target side is auto-managed.
let player = world.spawn(Player).id();
world.spawn((Enemy, Targeting(player)));
// world.entity(player).get::<Targets>() now contains the enemy.
```

Rules:

- Never mutate the target collection directly (it's read-only by design);
  insert/remove the relationship component instead.
- `commands.entity(e).with_related::<Targeting>(|spawner| ...)` and the
  `related!(Targets[...])` macro mirror `with_children`/`children![]`.
- Hierarchy-specific helpers: `add_child`, `insert_child`,
  `detach_child`/`detach_children`/`detach_all_children` (0.18 names;
  `remove_*`/`clear_*` pre-0.18), `despawn` cascades to children by default.
- Relationships currently model 1:many. Many:many needs manual modeling.

## EntityCommands / EntityWorldMut toolbox

```rust
commands.entity(e)
    .insert(CompA)
    .insert_if_new(CompB)        // don't overwrite existing
    .remove::<(CompA, CompB)>()  // bundles work for removal too
    .retain::<(Transform, Sprite)>()  // remove everything else
    .despawn();
commands.entity(e).entry::<Counter>().and_modify(|mut c| c.0 += 1).or_default();
let e2 = commands.entity(e).clone_and_spawn();   // duplicate an entity
commands.queue(|world: &mut World| { /* arbitrary deferred world access */ });
```

## Exclusive systems and direct World access

```rust
fn exclusive(world: &mut World) {
    let entity = world.spawn(Player).id();        // immediate, not deferred
    world.resource_scope(|world, mut score: Mut<Score>| {
        // access resource AND world simultaneously
    });
    let mut q = world.query_filtered::<&Transform, With<Player>>();
    for t in q.iter(world) {}
    // Run a one-shot system:
    let _ = world.run_system_cached(my_system);
}
```

Exclusive systems block all parallelism — keep them rare and fast. Also
available: `SystemState<P>` to use normal system params inside exclusive
code, caching it in a `Local` or resource for reuse.

### One-shot systems

```rust
let id = world.register_system(my_system);   // or commands.register_system
commands.run_system(id);                     // deferred run
// 0.18: commands.run_system_cached(my_system) avoids manual registration
```

Good for "call this logic from anywhere" (button callbacks, console
commands) without inventing a message type.

## Custom SystemParam

Bundle frequently co-occurring params into one named param:

```rust
#[derive(SystemParam)]
struct PlayerCtx<'w, 's> {
    players: Query<'w, 's, (&'static Transform, &'static Health), With<Player>>,
    score: Res<'w, Score>,
}
fn my_system(ctx: PlayerCtx) {}
```

## Entity disabling

Insert the `Disabled` component to hide an entity from queries without
despawning it (default query filters skip disabled entities; query for
`With<Disabled>` explicitly, or use `Allows<Disabled>`/`EntityRef` carefully,
to see them). Useful for object pooling and "soft delete". Custom disabling
components can be registered via `World::register_disabling_component`.

## Safe multi-component mutable access (0.18)

`EntityMut::get_components_mut::<(&mut A, &mut B)>()` returns multiple
mutable borrows with runtime aliasing checks — replaces unsafe juggling
when you need several components of one entity outside a query.

## Dynamic components/queries (advanced)

`ComponentDescriptor`-based dynamic components and `QueryBuilder` exist for
scripting/modding layers. Reach for them only when types aren't known at
compile time; check rustdoc for the current unsafe contracts.
