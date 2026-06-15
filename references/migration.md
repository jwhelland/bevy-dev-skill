# Version Migration Cheat Sheet (0.15 → 0.18)

Use this to (a) read older code/tutorials and translate to 0.18, or
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

## Upgrading a project: process

1. Bump one minor version at a time; read that release's migration guide.
2. `cargo check` and fix mechanically — most breakage is renames above.
3. Upgrade ecosystem crates in lockstep (each pins a Bevy minor;
   `cargo tree -d` to catch duplicates).
4. Watch runtime behavior changes that compile fine: state re-trigger
   semantics (0.18), `Option<Single>` multi-match (0.17), message
   double-buffer timing, despawn-recursive default (0.16).
