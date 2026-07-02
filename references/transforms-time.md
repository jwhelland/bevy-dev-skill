# Transforms, Hierarchy, Time, Timers

Targets Bevy 0.19.

## Transform vs GlobalTransform

- `Transform`: local — relative to parent (or world if no parent). You
  write this one.
- `GlobalTransform`: world-space, computed by transform propagation in
  `PostUpdate`. Read-only for you; convert with
  `global.translation()`, `.rotation()`, `.compute_transform()`.

```rust
Transform::from_xyz(1.0, 2.0, 3.0)
    .with_rotation(Quat::from_rotation_y(0.5))
    .with_scale(Vec3::splat(2.0));
Transform::from_translation(v).looking_at(target, Vec3::Y);
```

Manipulation:

```rust
fn orbit(mut q: Query<&mut Transform, With<Ship>>, time: Res<Time>) {
    for mut t in &mut q {
        t.translation += t.forward() * 5.0 * time.delta_secs(); // forward() = -local_z(), Dir3
        t.rotate_y(1.5 * time.delta_secs());                    // also rotate_local_y, rotate_axis
        t.rotation = t.rotation.slerp(target_rot, 0.1);         // smooth rotation
        t.look_at(Vec3::ZERO, Vec3::Y);
    }
}
```

Direction helpers on Transform: `forward/back/left/right/up/down()` (world
space `Dir3`). `Quat::from_rotation_arc(from, to)` builds a rotation
between directions. 2D rotation: `Quat::from_rotation_z(angle)`, or get
angle via `transform.rotation.to_euler(EulerRot::ZYX).0`.

## Hierarchy

Parenting via `ChildOf`/`Children` (see `ecs-advanced.md` relationships):

```rust
commands.spawn((Tank, Transform::default(), children![
    (Turret, Transform::from_xyz(0.0, 1.0, 0.0)),
]));
commands.entity(child).insert(ChildOf(parent));     // re-parent
commands.entity(parent).detach_children(&[child]);  // 0.18 name (was remove_children)
```

- A child's `Transform` is relative to its parent; scale/rotation inherit
  (non-uniform parent scale + child rotation = shear; avoid).
- Propagation timing: `GlobalTransform` is stale during `Update` if the
  parent moved this frame; if you need fresh values, run in `PostUpdate`
  `.after(TransformSystems::Propagate)`.
- Despawning a parent despawns children. Detach first if not wanted.
- World↔local conversion: `GlobalTransform::affine().inverse()` or
  `Transform::from_matrix(parent_global.affine().inverse() * child_global.affine())`.

## Time

```rust
fn sys(time: Res<Time>) {
    time.delta_secs();      // f32 seconds since last run of THIS schedule
    time.delta();           // Duration
    time.elapsed_secs();    // since app start
}
```

- `Res<Time>` is context-sensitive: in `FixedUpdate` it returns fixed
  timestep values automatically. Specific clocks: `Time<Real>` (unscaled
  wall time), `Time<Virtual>` (pausable/scalable), `Time<Fixed>`.
- Pause / slow motion:

```rust
fn pause(mut time: ResMut<Time<Virtual>>) {
    time.pause();              // unpause(), is_paused()
    time.set_relative_speed(0.5);
}
```

`Time<Virtual>` pause stops the default `Time` and `FixedUpdate`, but
systems still run — gate gameplay systems on a state too. `Time<Real>`
keeps running (use for UI animations during pause).

**Always multiply movement by delta** — never move per-frame constants.

## Timers and Stopwatches

`Timer`/`Stopwatch` are plain values — store them in components or
resources; tick them yourself.

```rust
#[derive(Component)]
struct AttackCooldown(Timer);   // Timer::from_seconds(0.5, TimerMode::Once) or Repeating

fn attack(time: Res<Time>, mut q: Query<&mut AttackCooldown>) {
    for mut cd in &mut q {
        cd.0.tick(time.delta());
        if cd.0.is_finished() {           // 0.17+: is_finished (was finished)
            cd.0.reset();
        }
    }
}
```

- `TimerMode::Repeating`: `just_finished()` true on completion frames;
  handles multiple completions per frame via `times_finished_this_tick()`.
- `TimerMode::Once`: stays finished until `reset()`; `is_finished()` stays
  true.
- Fractional progress for animation: `timer.fraction()` (0..1).
- One-off scheduled work: simplest is a component with a `Timer` + a system
  that acts and despawns/removes when finished ("delayed despawn" pattern).
- Frame-rate-independent exponential smoothing (camera follow):

```rust
let s = 1.0 - (-decay_rate * time.delta_secs()).exp();
cam.translation = cam.translation.lerp(target, s);
```

## Math quick notes

Bevy re-exports `glam`: `Vec2/Vec3/Quat/Mat4/Affine3A`, plus `Dir2/Dir3`
(guaranteed-normalized), `Rot2` (2D rotation), `Isometry2d/3d`
(translation+rotation), primitives (`Circle`, `Cuboid`, ...) with geometric
ops, `Rect`/`Aabb2d` etc. Curves/easing: `EasingCurve::new(a, b, EaseFunction::CubicOut)`
and the `Curve` trait; `f32::lerp`, `Vec3::lerp`, `Quat::slerp`,
`StableInterpolate::interpolate_stable`.
