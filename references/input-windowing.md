# Input and Windowing

Targets Bevy 0.18. Note: 0.18 split input sources behind features
(`keyboard`, `mouse`, `gamepad`, `touch`, `gestures`) — all on by default
with `DefaultPlugins`, relevant only for `default-features = false` builds.

## Keyboard & mouse buttons — ButtonInput

```rust
fn movement(keys: Res<ButtonInput<KeyCode>>, mouse: Res<ButtonInput<MouseButton>>) {
    if keys.pressed(KeyCode::KeyW) {}              // held
    if keys.just_pressed(KeyCode::Space) {}        // this frame only
    if keys.just_released(KeyCode::Escape) {}
    if keys.any_pressed([KeyCode::ShiftLeft, KeyCode::ShiftRight]) {}
    if mouse.just_pressed(MouseButton::Left) {}
}
```

`KeyCode` is the *physical* key (layout-independent — WASD stays WASD on
AZERTY). For typed characters, read `KeyboardInput` messages and use the
`Key`/`text` logical fields. Run-condition helpers exist:
`input_just_pressed(KeyCode::F5)` etc.

Movement vector idiom (normalize diagonals!):

```rust
let mut dir = Vec2::ZERO;
if keys.pressed(KeyCode::KeyW) { dir.y += 1.0; }
if keys.pressed(KeyCode::KeyS) { dir.y -= 1.0; }
if keys.pressed(KeyCode::KeyA) { dir.x -= 1.0; }
if keys.pressed(KeyCode::KeyD) { dir.x += 1.0; }
let velocity = dir.normalize_or_zero() * speed * time.delta_secs();
```

## Mouse motion, wheel, cursor

```rust
fn look(mut motion: MessageReader<bevy::input::mouse::MouseMotion>,
        mut wheel: MessageReader<bevy::input::mouse::MouseWheel>) {
    for m in motion.read() { /* m.delta: Vec2, raw mouse delta */ }
    for w in wheel.read() { /* w.y; check w.unit: Line vs Pixel (trackpads) */ }
}

// Cursor position (window space) and conversion to world:
fn cursor_world(window: Single<&Window>,
                camera: Single<(&Camera, &GlobalTransform), With<Camera2d>>) {
    let (cam, cam_tf) = *camera;
    if let Some(screen) = window.cursor_position() {
        if let Ok(world) = cam.viewport_to_world_2d(cam_tf, screen) { /* Vec2 */ }
        // 3D: cam.viewport_to_world(cam_tf, screen) -> Ray3d
    }
}
```

FPS-style mouse capture: with the 0.17+ window component split, grab
settings live on the `CursorOptions` component of the window entity:

```rust
fn grab(mut cursor: Single<&mut bevy::window::CursorOptions, With<PrimaryWindow>>) {
    cursor.grab_mode = bevy::window::CursorGrabMode::Locked;
    cursor.visible = false;
}
```

## Gamepad

Each connected gamepad is an **entity** with a `Gamepad` component:

```rust
fn gamepad(pads: Query<(Entity, &Gamepad)>) {
    for (_e, pad) in &pads {
        let move_x = pad.get(GamepadAxis::LeftStickX).unwrap_or(0.0);
        if pad.just_pressed(GamepadButton::South) { /* A / Cross */ }
        let rt = pad.get(GamepadButton::RightTrigger2).unwrap_or(0.0); // analog
    }
}
```

Connection events: `GamepadConnectionEvent` messages. Apply your own
deadzone for sticks (engine defaults are minimal): ignore
`stick.length() < 0.15`, rescale the rest. Rumble: write
`GamepadRumbleRequest` messages.

For bindable/rebindable input or input-as-actions across
keyboard+gamepad, use `leafwing-input-manager` (`ecosystem.md`) instead of
hand-rolling.

## Touch

`Res<Touches>`: `touches.iter()`, `just_pressed`, positions per touch id.
Gestures (pinch/rotate) arrive as messages on supporting platforms.

## Windows

The primary window is an entity with `Window` (+ split components since
0.17: `CursorOptions`, etc.). Configure at startup via `WindowPlugin` (see
SKILL.md) or mutate at runtime:

```rust
fn toggle_fullscreen(mut window: Single<&mut Window, With<PrimaryWindow>>,
                     keys: Res<ButtonInput<KeyCode>>) {
    if keys.just_pressed(KeyCode::F11) {
        use bevy::window::{WindowMode, MonitorSelection, VideoModeSelection};
        window.mode = match window.mode {
            WindowMode::Windowed => WindowMode::Fullscreen(MonitorSelection::Current,
                                                           VideoModeSelection::Current),
            _ => WindowMode::Windowed,
        };
    }
}
```

Useful `Window` fields: `resolution` (set size, `scale_factor`), `title`,
`present_mode` (`AutoVsync`/`AutoNoVsync` — the FPS-limiter), `resizable`,
`decorations`, `window_level`. Resize events: `WindowResized` messages.
Multiple windows: spawn another `Window` entity, point a camera's
`RenderTarget::Window(WindowRef::Entity(e))` at it.

Exit the app: write the `AppExit` message
(`fn quit(mut exit: MessageWriter<AppExit>) { exit.write(AppExit::Success); }`).
Close behavior: `WindowPlugin { exit_condition, close_when_requested, .. }`.

## Gotchas

- `just_pressed` is frame-scoped: reliable in `Update`, unreliable in
  `FixedUpdate` (read input in `Update`, forward into the fixed loop).
- Cursor position is `None` when the cursor is outside the window.
- Window-space Y is top-down; world Y is bottom-up — `viewport_to_world*`
  handles this; manual math must flip.
- On wasm, window size follows the canvas; set
  `Window { fit_canvas_to_parent: true, .. }`.
