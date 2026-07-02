# Assets: Loading, Handles, Custom Loaders

Targets Bevy 0.19.

## Basics

Assets live outside the ECS in `Assets<T>` collections, referenced by
`Handle<T>`. Loading is async; a handle is returned immediately, data
arrives later.

```rust
fn setup(mut commands: Commands, assets: Res<AssetServer>) {
    let image: Handle<Image> = assets.load("sprites/player.png");   // relative to assets/
    let scene: Handle<WorldAsset> = assets.load(GltfAssetLabel::Scene(0).from_asset("ship.glb"));
    commands.spawn(Sprite::from_image(image));
}
```

(`Scene` was renamed `WorldAsset` in 0.19 — `bevy_scene` → `bevy_world_serialization`, see `reflection-scenes.md`.)

- Paths are relative to the `assets/` folder next to `Cargo.toml` (or the
  executable in release). Sub-assets use labels: `"ship.glb#Scene0"` or the
  typed `GltfAssetLabel` API.
- `assets.load` deduplicates: same path → same handle.
- Handles are reference-counted; when the last **strong** handle drops, the
  asset unloads. Store handles in a resource to keep things loaded:

```rust
#[derive(Resource)]
struct UiAssets { font: Handle<Font>, icons: Vec<Handle<Image>> }
```

## Accessing and mutating asset data

```rust
fn tint(mut materials: ResMut<Assets<StandardMaterial>>, q: Query<&MeshMaterial3d<StandardMaterial>>) {
    for mat in &q {
        if let Some(m) = materials.get_mut(&mat.0) { m.base_color = Color::srgb(1.0, 0.0, 0.0); }
        // get_mut flags the asset modified → re-uploaded to GPU; affects ALL users of the handle.
        // 0.19: get_mut returns AssetMut<A> and only fires AssetEvent::Modified on an actual write
        // (0.18 fired it on every get_mut call, even a no-op borrow).
    }
}
```

Create assets at runtime with `materials.add(StandardMaterial { ... })` /
`meshes.add(Cuboid::default())`.

## Load state and "loading screen" pattern

```rust
fn check(assets: Res<AssetServer>, ui: Res<UiAssets>) {
    use bevy::asset::LoadState;
    match assets.load_state(&ui.font) {
        LoadState::Loaded => {}
        LoadState::Failed(err) => error!("{err}"),
        _ => {}   // NotLoaded / Loading
    }
}
```

Idiomatic loading screen: a `Loading` state that starts all loads
(`load_folder` for bulk → `Handle<LoadedFolder>`), a system polling
`assets.is_loaded_with_dependencies(&handle)` (or
`RecursiveDependencyLoadState`), transition to `Playing` when done. The
ecosystem crate `bevy_asset_loader` automates this well (see
`ecosystem.md`).

## Asset events

`AssetEvent<T>` is a **Message**:

```rust
fn watch(mut reader: MessageReader<AssetEvent<Image>>) {
    for ev in reader.read() {
        match ev {
            AssetEvent::LoadedWithDependencies { id } => {}
            AssetEvent::Modified { id } => {}   // fires on get_mut too
            AssetEvent::Added { id } | AssetEvent::Removed { id } | AssetEvent::Unused { id } => {}
        }
    }
}
```

Match `*id == handle.id()` to react to a specific asset.

## Custom asset types and loaders

```rust
#[derive(Asset, TypePath)]
struct Dialogue { lines: Vec<String> }

#[derive(Default, TypePath)]               // 0.18: loaders need TypePath
struct DialogueLoader;

impl AssetLoader for DialogueLoader {
    type Asset = Dialogue;
    type Settings = ();
    type Error = std::io::Error;

    async fn load(&self, reader: &mut dyn bevy::asset::io::Reader,
                  _settings: &(), _ctx: &mut bevy::asset::LoadContext<'_>)
                  -> Result<Dialogue, Self::Error> {
        let mut bytes = Vec::new();
        reader.read_to_end(&mut bytes).await?;
        Ok(Dialogue { lines: String::from_utf8_lossy(&bytes).lines().map(Into::into).collect() })
    }

    fn extensions(&self) -> &[&str] { &["dlg"] }
}

app.init_asset::<Dialogue>()
   .init_asset_loader::<DialogueLoader>();
```

Inside `load`, use `ctx.load(...)` to declare dependencies on other assets
and `ctx.labeled_asset_scope` to emit sub-assets. For settings-bearing
loaders, `Settings` must be serde + Default; per-asset overrides go in
`.meta` files or `assets.load_with_settings`.

## Embedded assets (airgapped/single-binary friendly)

```rust
// In a plugin:
embedded_asset!(app, "shaders/glow.wgsl");
// Load with the embedded:// scheme:
let shader = assets.load("embedded://my_crate/shaders/glow.wgsl");
```

Embeds the file into the binary at compile time — no assets folder needed.
Good for shaders and small core assets; for everything-embedded builds
consider `bevy_embedded_assets` (ecosystem).

## Hot reloading

```toml
bevy = { version = "0.19", features = ["file_watcher"] }
```

With the feature on (debug builds), editing a file under `assets/` reloads
it live and fires `AssetEvent::Modified`. Works for shaders, scenes,
images, custom assets. No code changes needed; react to Modified events if
derived state must rebuild.

## Asset processing (advanced, optional)

`bevy_asset`'s processor mode (`AssetPlugin { mode: AssetMode::Processed, .. }`)
runs assets through import pipelines at dev time (e.g. compress textures)
writing to `imported_assets/`. Most games don't need it yet; check rustdoc
(`bevy::asset::processor`) when you do.

## Pitfalls

- **Dropping all handles unloads the asset** — the classic "my texture
  disappears" bug. Keep a resource of handles.
- Asset paths are case-sensitive on Linux even if they work on
  macOS/Windows.
- `Assets::get_mut` triggers GPU re-upload for render assets — avoid
  calling it every frame; for per-entity variation prefer separate material
  handles or instanced data.
- Loading from outside `assets/` needs a custom `AssetSource` or absolute
  paths via `AssetPath`; sandboxed platforms (wasm, consoles) restrict this.
