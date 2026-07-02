# 3D Rendering: Meshes, PBR, Lights, Cameras, Custom Materials

Targets Bevy 0.19.

## Minimal 3D scene

```rust
fn setup(mut commands: Commands, mut meshes: ResMut<Assets<Mesh>>,
         mut materials: ResMut<Assets<StandardMaterial>>) {
    commands.spawn((
        Mesh3d(meshes.add(Cuboid::new(1.0, 1.0, 1.0))),
        MeshMaterial3d(materials.add(StandardMaterial {
            base_color: Color::srgb(0.8, 0.7, 0.6), ..default()
        })),
        Transform::from_xyz(0.0, 0.5, 0.0),
    ));
    commands.spawn((
        DirectionalLight { illuminance: 10_000.0, shadows_enabled: true, ..default() },
        Transform::from_rotation(Quat::from_euler(EulerRot::XYZ, -0.8, 0.4, 0.0)),
    ));
    commands.spawn((
        Camera3d::default(),
        Transform::from_xyz(-2.5, 4.5, 9.0).looking_at(Vec3::ZERO, Vec3::Y),
    ));
}
```

Primitive meshes: `Cuboid`, `Sphere`, `Plane3d`, `Cylinder`, `Capsule3d`,
`Torus`, `Cone` — all `meshes.add(...)`-able, with builder options
(`Sphere::new(1.0).mesh().ico(5)`).

## Loading glTF models

```rust
commands.spawn((
    WorldAssetRoot(assets.load(GltfAssetLabel::Scene(0).from_asset("ship.glb"))),
    Transform::from_xyz(0.0, 0.0, 0.0),
));
```

`SceneRoot` was renamed `WorldAssetRoot` in 0.19 (`bevy_scene` →
`bevy_world_serialization`, see `migration.md`/`reflection-scenes.md`). The
scene spawns as a hierarchy of child entities (named via `Name`). To modify
materials/parts after spawn, query children once loaded (observe
`SceneInstanceReady`) and match by `Name`. Typed access to gltf parts:
`Res<Assets<Gltf>>` → `gltf.named_meshes`, `.animations`, etc.
`GltfExtensionHandler` exists for custom glTF extension processing
(`on_material()` now receives `&GltfMaterial`).

**glTF materials load as `Handle<GltfMaterial>` by default in 0.19**, not
`Handle<StandardMaterial>`. Append `/std` to the asset label to get a
`StandardMaterial` handle instead: `"ship.glb#Material0/std"`.

## StandardMaterial (PBR) highlights

```rust
StandardMaterial {
    base_color: Color::WHITE,
    base_color_texture: Some(assets.load("albedo.png")),
    normal_map_texture: Some(assets.load("normal.png")),
    metallic: 0.0,            // 0 dielectric .. 1 metal
    perceptual_roughness: 0.5,
    emissive: LinearRgba::BLACK,
    alpha_mode: AlphaMode::Opaque,   // Blend / Mask(cutoff) / Premultiplied / Add
    double_sided: false, cull_mode: Some(Face::Back),
    ..default()
}
```

Transparency requires `alpha_mode: AlphaMode::Blend` *and* alpha < 1 in
`base_color`. Blend-mode meshes don't cast shadows by default and sort
per-mesh (expect artifacts on intersecting transparents).

## Lights

- `DirectionalLight` (sun; `illuminance` in lux, ~10k daylight) — usually
  one, with `shadows_enabled: true` and `CascadeShadowConfig` for quality.
- `PointLight` (`intensity` in lumens, e.g. 100_000 for a bright bulb;
  `range` matters for perf), `SpotLight`.
- Global ambient: `GlobalAmbientLight` **resource** (renamed from
  `AmbientLight` in 0.18); per-camera/entity `AmbientLight` component
  overrides it.
- Environment lighting (IBL): `EnvironmentMapLight` component on the camera
  with prefiltered ktx2 cubemaps; or `GeneratedEnvironmentMapLight`.
- Shadows cost per-light; limit shadow-casting lights. Per-entity opt-outs:
  `NotShadowCaster`, `NotShadowReceiver`.

## Camera & view effects

On the camera entity: `Camera { hdr: true, order, clear_color, .. }`,
`Projection::Perspective(...)`, `Tonemapping`, `Bloom`,
`ScreenSpaceAmbientOcclusion`, `TemporalAntiAliasing`/`Msaa`/`Fxaa`/`Smaa`,
`DepthOfField`, `MotionBlur`, `DistanceFog` (component `DistanceFog`),
`Skybox` (0.19: its `image` field is `Option<Handle<Image>>` — wrap
existing handles in `Some(...)`). `RenderTarget` is its own component
(render-to-texture: `RenderTarget::Image(image_handle.into())`).

**Procedural sky (`Atmosphere`) is a separate entity as of 0.19**, not a
camera component, and moved to `bevy::light::Atmosphere`:

```rust
use bevy::light::Atmosphere;

let earth = Atmosphere::earth(medium_handle);   // was Atmosphere::earthlike()
commands.spawn((earth, Transform::from_scale(Vec3::splat(0.001))));
commands.spawn((Camera3d::default(), AtmosphereSettings::default()));
```

`bottom_radius`/`top_radius` → `inner_radius`/`outer_radius`;
`AtmosphereSettings` dropped `scene_units_to_m` (use the atmosphere
entity's `Transform` scale instead). `ScatteringMedium` is still the asset
type for custom media.

0.18 ships `FreeCamera`/`FreeCameraPlugin` (fly camera) for quick scene
inspection.

Multiple cameras: set `Camera::order`; overlay cameras use
`clear_color: ClearColorConfig::None`. Split-screen via `Camera::viewport`.

## Visibility & performance

- `Visibility::Hidden` on an entity hides its subtree.
- Frustum culling is automatic (via `Aabb`, auto-updated in 0.18).
- Many identical meshes draw efficiently via automatic batching/instancing —
  share the same `Handle<Mesh>` + `Handle<StandardMaterial>`.
- Profile first: `cargo run --features bevy/trace_tracy` with Tracy, or
  `FrameTimeDiagnosticsPlugin` (see `testing-debugging.md`).

## Gizmos (immediate-mode debug drawing)

```rust
fn debug_draw(mut gizmos: Gizmos) {
    gizmos.cube(Isometry3d::IDENTITY, Vec3::ONE, Color::srgb(1.0, 0.0, 0.0)); // 0.18: cube (was cuboid)
    gizmos.line(Vec3::ZERO, Vec3::Y * 2.0, Color::WHITE);
    gizmos.sphere(Isometry3d::from_translation(Vec3::X), 0.5, Color::srgb(0.0, 1.0, 0.0));
    gizmos.arrow(Vec3::ZERO, Vec3::X, Color::srgb(1.0, 1.0, 0.0));
}
```

Redraw every frame (immediate mode). 2D variants exist (`gizmos.circle_2d`
etc.).

## Custom materials (WGSL shader)

```rust
use bevy::render::render_resource::AsBindGroup; // still here in 0.19
use bevy::shader::ShaderRef; // moved here from bevy::render::render_resource in 0.18 (E0432 if stale)

#[derive(Asset, TypePath, AsBindGroup, Clone)]
struct GlowMaterial {
    #[uniform(0)] color: LinearRgba,
    #[texture(1)] #[sampler(2)] tex: Option<Handle<Image>>,
}

impl Material for GlowMaterial {
    fn fragment_shader() -> ShaderRef { "shaders/glow.wgsl".into() }
    fn alpha_mode(&self) -> AlphaMode { AlphaMode::Blend }
}
// 0.19: AsBindGroup now requires a `fn label()` too (derive provides a default from the type name).

app.add_plugins(MaterialPlugin::<GlowMaterial>::default());
// spawn with MeshMaterial3d(glow_materials.add(GlowMaterial { ... }))
```

WGSL side imports Bevy's mesh bindings:

```wgsl
#import bevy_pbr::forward_io::VertexOutput
@group(#{MATERIAL_BIND_GROUP}) @binding(0) var<uniform> color: vec4<f32>;
@fragment
fn fragment(in: VertexOutput) -> @location(0) vec4<f32> { return color; }
```

To extend (not replace) PBR shading use `ExtendedMaterial<StandardMaterial, MyExt>`
with `MaterialExtension`. Post-processing in 0.18: `FullscreenMaterial`
trait + `FullscreenMaterialPlugin`. Embed shaders with `embedded_asset!`
for airgapped/single-binary builds (`assets.md`).

## Render-world internals (when patterns above aren't enough)

Custom render features involve: `ExtractComponentPlugin`/extract systems
(copy data main→render world), `RenderApp` sub-app, and render phases. In
0.19 the render graph was replaced internally by ordinary ECS
systems/schedules — custom render-graph nodes need updating (custom phases
now use a change-list system and `DirtySpecializations`), but
`Material`/`MaterialExtension`/`FullscreenMaterial` code is unaffected.
This is deep water either way — read the rustdoc for `bevy_render` and the
repo examples under `examples/shader/` and `examples/3d/`. Avoid it unless
those higher-level APIs truly can't express the effect.

## 3D gotchas

- Black scene = no light, or light pointing away; washed out = duplicate
  lights or tonemapping changes.
- Coordinate system: right-handed, +Y up, camera looks **-Z** ("forward" is
  `-Transform::local_z()` or `transform.forward()`).
- glTF imports +Z-forward models facing the wrong way; 0.18 has
  `GltfConvertCoordinates` options on the loader.
- HDR/bloom: bloom needs `Camera { hdr: true, .. }`.
- Shadow acne / peter-panning: tune `DirectionalLight` `shadow_depth_bias`
  / `shadow_normal_bias`.
