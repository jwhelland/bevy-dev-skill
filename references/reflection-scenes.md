# Reflection, Scenes, Serialization

Targets Bevy 0.18.

## Reflect

Bevy's runtime reflection powers scenes, the inspector crates, animation of
arbitrary fields, and (eventually) the editor. Derive it on data you want
serialized/inspectable, and register the type:

```rust
#[derive(Component, Reflect, Default)]
#[reflect(Component)]            // registers the ReflectComponent "type data"
struct Health { current: f32, max: f32 }

#[derive(Resource, Reflect, Default)]
#[reflect(Resource)]
struct Settings { volume: f32 }

app.register_type::<Health>()
   .register_type::<Settings>();
```

Rules of thumb:

- `#[reflect(Component)]` / `#[reflect(Resource)]` are required for scene
  round-tripping; plain `Reflect` alone isn't enough.
- Skip a field: `#[reflect(ignore)]` (must then be `Default`-constructible);
  opaque types: `#[reflect(opaque)]`.
- Most Bevy built-ins are already registered.
- 0.18: attribute syntax is parentheses only — `#[reflect(Clone)]`.

Dynamic use (rarely needed in game code, core to tooling):

```rust
let dyn_ref: &dyn Reflect = &health;
if let Some(cur) = health.reflect_path::<f32>(".current").ok() {}
// Type registry lookups:
let registry = app.world().resource::<AppTypeRegistry>().read();
```

## Scenes

A `Scene`/`DynamicScene` is a serialized set of entities + components (and
resources). Bevy's scene format is `.scn.ron`.

### Spawning

```rust
commands.spawn(DynamicSceneRoot(assets.load("levels/level1.scn.ron")));
// glTF scenes use SceneRoot (rendering-3d.md)
```

Scene contents spawn as *children* of the root entity; despawn the root to
unload the level. React to load completion by observing
`SceneInstanceReady` on the root:

```rust
commands.spawn(DynamicSceneRoot(handle))
    .observe(|ev: On<SceneInstanceReady>, q: Query<&Children>| { /* post-process */ });
```

### Creating / saving (dev-time tooling)

```rust
fn save(world: &World) {
    let scene = DynamicSceneBuilder::from_world(world)
        .allow_component::<Health>()
        .allow_component::<Transform>()
        .extract_entities(world.iter_entities().map(|e| e.id()))
        .build();
    let registry = world.resource::<AppTypeRegistry>().read();
    let ron = scene.serialize(&registry).unwrap();
    // Write to assets/levels/... (native only; use IoTaskPool for async file IO)
}
```

Only registered types with `#[reflect(Component)]` serialize. Entity ids in
files are placeholders; references remap on spawn.

### Example scene file

```ron
(
  resources: {},
  entities: {
    4242: (
      components: {
        "my_game::Health": (current: 10.0, max: 10.0),
        "bevy_transform::components::transform::Transform": (
          translation: (x: 0.0, y: 0.0, z: 0.0),
          rotation: (x: 0.0, y: 0.0, z: 0.0, w: 1.0),
          scale: (x: 1.0, y: 1.0, z: 1.0),
        ),
      },
    ),
  },
)
```

Hot reloading (`file_watcher` feature) makes `.scn.ron` files live-editable.

Note: a next-gen scene system ("BSN") has been in development; 0.18 still
uses the format above. Check release notes when upgrading past 0.18.

## Save games & general serialization

Scenes are a *possible* save format but awkward for full save games
(entity remapping, version drift). Common pragmatic approach:

- Define plain serde structs for persistent state (player progress,
  inventory, settings) decoupled from ECS layout.
- Systems translate ECS ↔ save structs; write with `serde_json`/`ron`/
  `postcard` (add as direct deps — bevy no longer re-exports ron).
- Settings/config: load at startup into a resource.
- The `moonshine-save` / `bevy_persistent` ecosystem crates formalize this.

For network replication, see `bevy_replicon` (`ecosystem.md`).

## Gotchas

- "Type not registered" panic/log when spawning a scene → missing
  `app.register_type::<T>()` or missing `#[reflect(Component)]`.
- Components with non-Reflect fields need `#[reflect(ignore)]` + sensible
  `Default`, or `#[reflect(opaque)]`.
- `Handle<T>` fields serialize as asset paths only when the handle has a
  path; runtime-generated assets don't round-trip.
- File writing is native-only; wasm saves go to localStorage via an
  ecosystem crate (e.g. `bevy_pkv`).
