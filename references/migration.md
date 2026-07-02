# Version Migration Cheat Sheet (0.15 → 0.19)

Use this to (a) read older code/tutorials and translate to 0.19, or
(b) write code for a project pinned to an older Bevy. Full per-release
guides: bevy.org/learn/migration-guides (offline: each release's guide is
also summarized in rustdoc release notes; this file covers the
high-traffic items).

## Quick version identification

| If you see... | The code targets |
|---|---|
| `SpriteBundle`, `Camera2dBundle`, `TextBundle` | ≤0.14 (bundles removed in 0.15) |
| `Sprite` + required components, `OnAdd` observers, `Trigger<E>` | 0.15–0.16 |
| `EventReader`/`EventWriter` for buffered events | ≤0.16 |
| `MessageReader`/`MessageWriter`, `On<E>` | 0.17+ |
| `children![]`, `related!`, `Single` param | 0.16+ |
| `detach_children`, `GlobalAmbientLight`, `get_disjoint_mut` | 0.18 |
| `SceneRoot`/`DynamicSceneRoot`, `bevy_scene::Scene` | ≤0.18 (scene system renamed in 0.19) |
| `WorldAssetRoot`, `bevy_world_serialization`, `TextFont.font_size: FontSize`, separate `Component`/`Resource` derives | 0.19 |

## 0.14 → 0.15 (required components revolution)

- Bundles deprecated as the main spawn API. `SpriteBundle` →
  `Sprite` (+ auto-required `Transform`, `Visibility`);
  `Camera2dBundle` → `Camera2d`; `Camera3dBundle` → `Camera3d`;
  `PbrBundle` → `Mesh3d(handle)` + `MeshMaterial3d(handle)`;
  `SpriteSheetBundle` → `Sprite::from_atlas_image`;
  `Text2dBundle` → `Text2d`; `NodeBundle`/`Style` → `Node` (style fields
  moved into `Node`; old `Node` → `ComputedNode`);
  `TextBundle` → `Text`, with `TextFont`/`TextColor` components.
- `Handle<Image>`-as-component removed: use `Sprite`/`ImageNode` wrappers.
- Observers/hooks stabilized; `Commands::trigger` etc.
- `Query::single()` started returning Result later (0.16/0.17 hardening).

## 0.15 → 0.16

- **Relationships**: `Parent` component → `ChildOf` (field access
  `child_of.parent()`); custom relationships via `#[relationship]`.
  `despawn_recursive()` → plain `despawn()` (now recursive by default);
  `despawn_descendants()` → `despawn_related::<Children>()`.
- `children![]` / `related!` spawn macros; `Commands::spawn` with
  `SpawnRelated`.
- GPU-driven rendering, `Single`/`Populated` params widely used.
- Error handling: systems may return `Result`; `query.single()` returns
  `Result` (was panic).
- `Trigger::entity()` → `trigger.target()` (0.16 naming).
- States moved along; `StateScoped(state)` for auto-despawn.

## 0.16 → 0.17 (the big rename)

Buffered events became **Messages**:

| 0.16 | 0.17+ |
|---|---|
| `#[derive(Event)]` (buffered) | `#[derive(Message)]` |
| `EventReader<E>` / `EventWriter<E>` | `MessageReader<M>` / `MessageWriter<M>` |
| `Events<E>` | `Messages<M>` |
| `app.add_event::<E>()` | `app.add_message::<M>()` |
| `writer.send(e)` / `commands.send_event(e)` | `writer.write(m)` / `commands.write_message(m)` |

Observer events:

| 0.16 | 0.17+ |
|---|---|
| `Trigger<E>` param | `On<E>` |
| `OnAdd`, `OnInsert`, `OnRemove`, `OnReplace`, `OnDespawn` | `Add`, `Insert`, `Remove`, `Replace`, `Despawn` |
| `trigger.target()` (observer target entity) | `EntityEvent` trait; event carries target field; `On::event_target()` / event's `entity` field |
| `trigger_targets(event, entities)` | `EntityEvent` with entity in the event struct |

Other 0.17 changes:

- `StateScoped` → `DespawnOnExit` (+ `DespawnOnEnter`).
- `Window` split into components (`CursorOptions`, etc.).
- Set names: `RenderSet`→`RenderSystems`, `UiSystem`→`UiSystems`,
  `TransformSystem`→`TransformSystems`, `PickSet`→`PickingSystems`;
  `Condition` trait → `SystemCondition`.
- `Timer::finished()`→`is_finished()`, `paused()`→`is_paused()`.
- `JustifyText` → `Justify`; camera/visibility types moved to
  `bevy_camera`; `Handle::Weak` removed (`Handle::Uuid`/`uuid_handle!`).
- `Option<Single<..>>` now `None` when **multiple** match too.
- Feathers/widgets introduced (experimental); `bevy_ui_widgets` headless.

## 0.17 → 0.18

- `get_many_mut` → `get_disjoint_mut` (and `_unchecked` variant).
- Children: `remove_children`→`detach_children`, `remove_child`→
  `detach_child`, `clear_children`→`detach_all_children`,
  `clear_related`→`detach_all_related`.
- `AmbientLight` resource → `GlobalAmbientLight` (component form remains as
  per-camera override).
- `RenderTarget` is its own component on cameras (was `Camera` field).
- `NextState::set` now re-triggers transitions even for the same state;
  `set_if_neq` restores old behavior.
- `Gizmos::cuboid` → `Gizmos::cube`.
- `BorderRadius` component → `Node.border_radius` field; `LineHeight` out
  of `TextFont` into its own component.
- `Entity::row()` → `Entity::index()`; entity allocator API reworked
  (`EntitiesAllocator`).
- Asset loaders/transformers/savers/process require `TypePath`;
  `LoadContext::asset_path()` → `path()`; asset channels are
  `async_channel` (`send_blocking`); `AssetSource::build()` →
  `AssetSourceBuilder::new(...)`.
- `Material`: `MaterialPlugin { prepass_enabled, shadows_enabled }` fields →
  trait methods `enable_prepass()`/`enable_shadows()`; `AsBindGroup` now
  requires `fn label()`.
- `#[reflect(...)]` parentheses-only syntax.
- Aabbs auto-update (delete manual recompute workarounds; `NoAutoAabb` to
  opt out).
- Picking feature renames: `bevy_sprite_picking_backend`→`sprite_picking`,
  `bevy_ui_picking_backend`→`ui_picking`, `bevy_mesh_picking_backend`→`mesh_picking`;
  `animation` feature → `gltf_animation`.
- New feature collections: `2d`, `3d`, `ui`; input device features
  (`keyboard`, `mouse`, `gamepad`, ...) must be explicit in
  `default-features = false` builds.
- `AnimationTarget {id, player}` split → `AnimationTargetId` + `AnimatedBy`.
- `SimpleExecutor` removed; `ron` re-exports removed (depend on `ron`
  directly); `FontAtlasSets` removed (use `FontAtlasSet`).
- glTF: `use_model_forward_direction` → `convert_coordinates:
  GltfConvertCoordinates`.

## 0.18 → 0.19

The biggest release in a while — architectural changes under the hood, plus
several everyday-API renames.

**Resources are now components** (the headline change): `#[derive(Resource)]`
implements `Component` too, and resources live as components on dedicated
singleton entities.

- You can no longer doubly derive: `#[derive(Component, Resource)]` on one
  type is a compile error. Split into two types (a `Comp` and a `Res`) if
  you relied on that.
- Broad queries like `Query<()>` or `Query<EntityMut>` now also match the
  hidden resource entities and conflict with `Res`/`ResMut` access on the
  same type. Add `Without<IsResource>` (or a `Without<MyResource>` filter)
  to exclude them.
- Non-send API renamed to match: `App::init_non_send_resource()` →
  `init_non_send()`, `App::insert_non_send_resource()` → `insert_non_send()`,
  `World::non_send_resource()`/`_mut()` → `World::non_send()`/`non_send_mut()`.
- `#[derive(MapEntities)]` is no longer needed on resources — implemented by
  default.

**Scenes reorganized to make room for BSN.** The old runtime scene
(`.scn.ron`) system moved to a new crate, freeing up `bevy_scene` for
**BSN** (Bevy Scene Notation), Next Generation Scenes' code-first `bsn!`
macro (a `.bsn` asset loader isn't shipping yet — BSN today is a
code-driven workflow, e.g. `bsn! { Player { score: 0 } Team::Blue }`).

| 0.18 (`bevy_scene`) | 0.19 (`bevy_world_serialization`) |
|---|---|
| `Scene` | `WorldAsset` |
| `SceneRoot` | `WorldAssetRoot` |
| `DynamicScene` | `DynamicWorld` |
| `DynamicSceneRoot` | `DynamicWorldRoot` |
| `DynamicSceneBuilder` | `DynamicWorldBuilder` (now takes `&TypeRegistry`) |
| `SceneSpawner` | `WorldInstanceSpawner` |
| `ScenePlugin` | `WorldSerializationPlugin` |
| `SceneLoader` | `WorldAssetLoader` |
| `SceneFilter` | `WorldFilter` |

glTF scene spawning uses the new name too:

```rust
// 0.18
commands.spawn(SceneRoot(assets.load("ship.glb#Scene0")));
// 0.19
commands.spawn(WorldAssetRoot(assets.load("ship.glb#Scene0")));
```

**Text overhaul**: text internals moved from `cosmic-text` to `parley`.
`TextFont` field types changed:

```rust
// 0.18
TextFont { font: assets.load("font.ttf"), font_size: 24.0, ..default() }
// 0.19
TextFont { font: assets.load("font.ttf").into(), font_size: FontSize::Px(24.0), ..default() }
```

`font: Handle<Font>` → `FontSource` (call `.into()` on a handle);
`font_size: f32` → `FontSize` (wrap in `FontSize::Px(...)`).
`TextLayout::new_with_justify()`/`new_with_linebreak()`/`new_with_no_wrap()`
→ `TextLayout::justify()`/`linebreak()`/`no_wrap()`.

**Atmosphere restructured**: moved to `bevy::light::Atmosphere`, spawned as
its own entity instead of a camera component, and radii renamed:

```rust
// 0.18
commands.spawn((Camera3d::default(), Atmosphere::earthlike(medium)));
// 0.19
use bevy::light::Atmosphere;
let earth = Atmosphere::earth(medium);
commands.spawn((earth, Transform::from_scale(Vec3::splat(0.001))));
commands.spawn((Camera3d::default(), AtmosphereSettings::default()));
```

`Atmosphere::earthlike()` → `Atmosphere::earth()`; `bottom_radius`/
`top_radius` → `inner_radius`/`outer_radius`; `AtmosphereSettings` dropped
`scene_units_to_m` (use `Transform` scale on the atmosphere entity instead).

**Feathers/widgets stabilized**: `experimental_bevy_feathers` feature →
`bevy_feathers`; `FeathersPlugin` → `FeathersCorePlugin`;
`experimental_ui_widgets` → `bevy_ui_widgets` (now part of the `ui` feature
collection); `UiWidgetsPlugins`/`InputDispatchPlugin` are now in
`DefaultPlugins`. Widget component prefixes dropped: `CoreScrollbarThumb` →
`ScrollbarThumb`, `CoreSliderDragState` → `SliderDragState`. `InputFocus`'s
`.0` field is private now — use `.get()`/`.set(entity)`/`.clear()`.

**Audio feature decoupled**: `audio` is no longer implied by `2d`/`3d`/`ui`
— enable it explicitly in slimmed `default-features = false` builds. Rodio
bumped to 0.22; format features renamed/split: `vorbis`, `wav`, `mp3`,
`mp4`, `flac`, `aac`, plus `symphonia-*` backend features and an
`audio-all-formats` convenience collection.

**Misc renames worth knowing**:

- `ShaderStorageBuffer` → `ShaderBuffer`.
- `Font::try_from_bytes()` → `Font::from_bytes()` (no longer returns `Result`).
- `Assets::get_mut()` now returns `AssetMut<A>` and only fires
  `AssetEvent::Modified` on an actual mutation (was: every call).
- `AssetPath::resolve(&str)` → takes `&AssetPath` now; use `resolve_str()`
  for a plain string argument (same split for `resolve_embed`).
- glTF materials load as `Handle<GltfMaterial>` by default now, not
  `Handle<StandardMaterial>`; append `/std` to the asset label
  (`"model.glb#Material0/std"`) to get a `StandardMaterial` handle.
- Render graph replaced by ECS systems/schedules internally — only matters
  if you wrote custom render-graph nodes; `Material`/`MaterialExtension`/
  `FullscreenMaterial` users are unaffected.
- Bloom's luma calculation moved to linear color space — subtly different
  look at the same settings, not a code change.

## Upgrading a project: process

1. Bump one minor version at a time; read that release's migration guide.
2. `cargo check` and fix mechanically — most breakage is renames above.
3. Upgrade ecosystem crates in lockstep (each pins a Bevy minor;
   `cargo tree -d` to catch duplicates).
4. Watch runtime behavior changes that compile fine: state re-trigger
   semantics (0.18), `Option<Single>` multi-match (0.17), message
   double-buffer timing, despawn-recursive default (0.16), broad
   `Query<()>`/`EntityMut` queries silently picking up resource entities
   (0.19 — filter with `Without<IsResource>`).
