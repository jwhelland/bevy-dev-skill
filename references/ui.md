# UI: Nodes, Layout, Widgets, Picking

Targets Bevy 0.19. Bevy UI is ECS-based: every UI element is an entity with
a `Node` component; layout is flexbox/grid (taffy). `bevy_feathers` is the
first-party widget/theme set, stabilized (no longer `experimental_*`) in
0.19 — for heavy editor-style UI consider `bevy_egui`, see `ecosystem.md`.

## Core structure

```rust
fn setup_ui(mut commands: Commands, assets: Res<AssetServer>) {
    commands.spawn((
        Node {                                     // root: covers the window
            width: Val::Percent(100.0),
            height: Val::Percent(100.0),
            justify_content: JustifyContent::Center,  // main axis
            align_items: AlignItems::Center,          // cross axis
            flex_direction: FlexDirection::Column,
            row_gap: Val::Px(12.0),
            ..default()
        },
        children![
            (
                Text::new("Hello"),
                TextFont { font: assets.load("fonts/Mono.ttf").into(), font_size: FontSize::Px(32.0), ..default() },
                TextColor(Color::WHITE),
            ),
            (
                Node { width: Val::Px(220.0), height: Val::Px(56.0),
                       justify_content: JustifyContent::Center, align_items: AlignItems::Center,
                       border: UiRect::all(Val::Px(2.0)), ..default() },
                BackgroundColor(Color::srgb(0.15, 0.15, 0.2)),
                BorderColor::all(Color::WHITE),
                Button,
                children![Text::new("Play")],
            ),
        ],
    ));
}
```

- UI needs a camera (any `Camera2d`/`Camera3d`); UI renders per-camera —
  multi-camera setups can scope UI with `UiTargetCamera`.
- Coordinates: top-left origin, logical pixels.
- `Val`: `Px`, `Percent`, `Vw/Vh/VMin/VMax`, `Auto`.
- Styling components alongside `Node`: `BackgroundColor`, `BorderColor`,
  `Outline`, `BoxShadow`, `ZIndex`/`GlobalZIndex`. `Node.border_radius`
  is a field on `Node` in 0.18 (was a separate `BorderRadius` component).
- `Display::Grid` for grid layout (`grid_template_columns`, etc.);
  `Display::None` removes from layout (vs `Visibility::Hidden` which keeps
  space).

## Interaction: two generations

**Classic polling** (simple, fine for buttons):

```rust
fn buttons(mut q: Query<(&Interaction, &mut BackgroundColor),
                       (Changed<Interaction>, With<Button>)>) {
    for (i, mut bg) in &mut q {
        match i {
            Interaction::Pressed => bg.0 = Color::srgb(0.3, 0.3, 0.5),
            Interaction::Hovered => bg.0 = Color::srgb(0.2, 0.2, 0.35),
            Interaction::None => bg.0 = Color::srgb(0.15, 0.15, 0.2),
        }
    }
}
```

**Picking events + observers** (modern, scales better; needs `ui_picking`
feature, on by default):

```rust
commands.spawn((Button, Node { ..default() }))
    .observe(|click: On<Pointer<Click>>, mut next: ResMut<NextState<GameState>>| {
        next.set(GameState::Playing);
    });
```

Pointer events: `Over`, `Out`, `Press`, `Release`, `Click`, `Move`,
`Drag`, `DragStart`, `DragEnd`, `DragEnter`, `Scroll`... They propagate up
the hierarchy (call `click.propagate(false)` to stop). The same picking
model serves sprites (`sprite_picking`) and 3D meshes (`mesh_picking`).
`Pickable` component tunes per-entity behavior (e.g.
`Pickable::IGNORE`).

## Text

`Text` (UI) / `Text2d` (world). Styling per-entity via `TextFont`,
`TextColor`, `TextLayout` (justify, linebreak). Rich text = child
`TextSpan` entities, each with their own font/color. To mutate text:

**0.19 text overhaul** (`cosmic-text` → `parley`): `TextFont.font` is now
`FontSource` (`handle.into()`), `TextFont.font_size` is now `FontSize`
(`FontSize::Px(32.0)`; also supports viewport/rem-relative units).
`TextLayout::new_with_justify()`/`new_with_linebreak()`/`new_with_no_wrap()`
→ `TextLayout::justify()`/`linebreak()`/`no_wrap()`. New `EditableText`
component gives upstream text-input support (IME, multiline, Unicode-aware)
— see the `text_input` example.

```rust
fn update_score(score: Res<Score>, mut q: Query<&mut Text, With<ScoreLabel>>) {
    if score.is_changed() {
        for mut t in &mut q { t.0 = format!("Score: {}", score.0); }
    }
}
```

0.18 adds `Underline`/`Strikethrough` components, `FontWeight` (variable
fonts), `FontFeatures` (OpenType), and per-span picking (hyperlinks).

## Images, scrolling, misc

- `ImageNode::new(handle)` for UI images (atlas support included).
- Scrolling: `Node { overflow: Overflow::scroll_y(), .. }` + write to
  `ScrollPosition`; mouse-wheel scrolling must be wired by you or use
  feathers widgets. 0.18: `IgnoreScroll` lets a child opt out.
- `RelativeCursorPosition` component gives the cursor pos within a node.

## bevy_feathers / standard widgets

Feathers (feature `bevy_feathers`, stabilized from `experimental_bevy_feathers`
in 0.19) provides themed widgets built on `bevy_ui_widgets` headless
behaviors (feature `bevy_ui_widgets`, formerly `experimental_ui_widgets` —
now part of the `ui` feature collection, and its plugins
(`UiWidgetsPlugins`, `InputDispatchPlugin`) ship in `DefaultPlugins`):
buttons, sliders, checkbox, radio (`RadioButton`/`RadioGroup`), color
pickers (`ColorPlane`), `Popover`, `MenuPopup`, virtual keyboard, and (0.19)
text input/dropdowns/list views. Plugin renamed: `FeathersPlugin` →
`FeathersCorePlugin`. Widget component names dropped their `Core` prefix:
`CoreScrollbarThumb` → `ScrollbarThumb`, `CoreSliderDragState` →
`SliderDragState`. Some spawning helper functions were renamed too (e.g.
`button` → `button_bundle`); check `cargo doc -p bevy_feathers` for the
exact constructors in your pinned version before writing code against it.
For shipping games, custom-styled `Button`+observers or egui are still
often simpler.

Directional (gamepad/keyboard) navigation: `AutoDirectionalNavigation` for
automatic focus graphs; `InputFocus` resource tracks focus — as of 0.19 its
inner `Entity` field is private, use `.get()` / `.set(entity)` / `.clear()`
instead of `.0`.

## Layout debugging

- `bevy_dev_tools` feature → `UiDebugOptions` resource: toggle overlay
  (`options.toggle()`) to outline all nodes.
- Common invisible-UI causes: zero-size root (give the root explicit
  width/height), missing camera, `GlobalZIndex` battles, text empty,
  font asset path wrong.
- Common layout confusion: `justify_content` works on the main axis which
  *depends on `flex_direction`*; `align_items` is the cross axis. `Auto`
  margins absorb free space (centering trick: `margin: UiRect::AUTO`).
