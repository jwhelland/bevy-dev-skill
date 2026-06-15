# 2D Rendering: Sprites, Atlases, Text2d, Tilemaps

Targets Bevy 0.18.

## Camera

```rust
commands.spawn(Camera2d);    // required components pull in Camera, Transform, etc.
```

- 2D camera looks down -Z; world units = pixels by default (1 unit = 1
  logical pixel at scale 1).
- Zoom/scale via `Projection::Orthographic`:

```rust
commands.spawn((
    Camera2d,
    Projection::Orthographic(OrthographicProjection {
        scaling_mode: bevy::camera::ScalingMode::FixedVertical { viewport_height: 1080.0 },
        ..OrthographicProjection::default_2d()
    }),
));
```

`ScalingMode` options: `WindowSize` (1:1 pixels), `FixedVertical`/
`FixedHorizontal` (consistent world view across resolutions), `AutoMin`/
`AutoMax`. For pixel art also set `ImagePlugin::default_nearest()` on
DefaultPlugins.

- 0.18 ships `PanCamera` + `PanCameraPlugin` for ready-made 2D pan/zoom
  controls (dev tooling; fine to use in-game too).

## Sprites

```rust
commands.spawn((
    Sprite {
        image: assets.load("player.png"),
        color: Color::WHITE,                    // tint (multiply)
        flip_x: false,
        custom_size: Some(Vec2::new(64.0, 64.0)),  // override image size
        anchor: bevy::sprite::Anchor::Center,
        ..default()
    },
    Transform::from_xyz(0.0, 0.0, 1.0),         // Z = draw order (higher = on top)
));
// Shorthand: Sprite::from_image(handle)
```

- **Z-ordering**: 2D sorts by global Z translation. Reserve Z bands per
  layer (background 0, world 10, actors 20, fx 30, ...).
- Visibility: `Visibility::Visible/Hidden/Inherited` component;
  `ViewVisibility` is the computed result.
- 9-slice / tiled sprites: `Sprite { image_mode: SpriteImageMode::Sliced(TextureSlicer { .. }), .. }`.

## Texture atlases / sprite animation

```rust
let layout = layouts.add(TextureAtlasLayout::from_grid(
    UVec2::splat(24), 7, 1, None, None));   // tile size, columns, rows, padding, offset

commands.spawn((
    Sprite::from_atlas_image(assets.load("run.png"),
        TextureAtlas { layout, index: 0 }),
    AnimTimer(Timer::from_seconds(0.1, TimerMode::Repeating)),
));

fn animate(time: Res<Time>, mut q: Query<(&mut AnimTimer, &mut Sprite)>) {
    for (mut t, mut sprite) in &mut q {
        if t.0.tick(time.delta()).just_finished() {
            if let Some(atlas) = sprite.texture_atlas.as_mut() {
                atlas.index = (atlas.index + 1) % 7;
            }
        }
    }
}
```

## 2D meshes and materials

For shapes/custom shaders in 2D:

```rust
commands.spawn((
    Mesh2d(meshes.add(Circle::new(50.0))),
    MeshMaterial2d(materials.add(ColorMaterial::from(Color::srgb(0.2, 0.7, 1.0)))),
    Transform::from_xyz(0.0, 0.0, 5.0),
));
```

Custom 2D shader: implement `Material2d` (same shape as `Material`, see
`rendering-3d.md`) and add `Material2dPlugin::<MyMaterial>::default()`.

## Text in world space

```rust
commands.spawn((
    Text2d::new("damage!"),
    TextFont { font: assets.load("fonts/Mono.ttf"), font_size: 24.0, ..default() },
    TextColor(Color::srgb(1.0, 0.3, 0.3)),
    Transform::from_xyz(0.0, 40.0, 50.0),
));
```

UI-space text is `Text` (see `ui.md`). Multi-style runs use child
`TextSpan` entities.

## Tilemaps

Bevy has no first-party tilemap renderer. Options:

- **Naive**: one sprite entity per tile. Fine to a few thousand tiles;
  simple, works with all tooling.
- **`bevy_ecs_tilemap`** (ecosystem): chunked GPU tilemap rendering, huge
  maps, Tiled/LDtk loaders exist on top of it (`bevy_ecs_ldtk`). Use this
  for real tile games (see `ecosystem.md`).

## Picking (clicking sprites)

Sprite picking ships behind the `sprite_picking` feature; with it enabled,
sprites get pointer events via observers:

```rust
commands.spawn(Sprite::from_image(tex))
    .observe(|click: On<Pointer<Click>>| { info!("clicked {}", click.entity); });
```

(Full picking model: `ui.md`.) For gameplay click-to-world, the manual
route is `camera.viewport_to_world_2d(cam_transform, cursor_pos)` — see
`input-windowing.md` for cursor position.

## 2D gotchas

- A sprite not appearing usually means: no `Camera2d`; Z behind another
  sprite or outside near/far (default 2D camera sees Z ≈ -1000..1000);
  asset path wrong (check logs); or transform scale 0.
- `Aabb` culling bounds auto-update in 0.18 (no more stale-bounds bug after
  resizing sprites; opt out with `NoAutoAabb`).
- Mixing UI (`Node`) and world (`Sprite`) coordinate spaces: UI origin is
  top-left in logical pixels; world 2D origin is center with +Y up.
