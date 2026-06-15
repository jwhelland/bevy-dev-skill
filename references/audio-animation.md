# Audio and Animation

Targets Bevy 0.18.

## Audio

Built-in audio (`bevy_audio`, rodio-backed) is simple playback. For mixing
buses, effects chains, ducking → use `bevy_kira_audio` or `bevy_seedling`
(`ecosystem.md`).

```rust
// Fire-and-forget sound effect:
commands.spawn((
    AudioPlayer::new(assets.load("sfx/jump.ogg")),
    PlaybackSettings::DESPAWN,    // despawn entity when done
));

// Looping music with volume:
commands.spawn((
    AudioPlayer::new(assets.load("music/theme.ogg")),
    PlaybackSettings::LOOP.with_volume(bevy::audio::Volume::Linear(0.4)),
));
```

- `PlaybackSettings` presets: `ONCE`, `LOOP`, `DESPAWN`, `REMOVE`; options:
  `.with_volume`, `.with_speed`, `.paused`, `.with_spatial(true)`.
- Runtime control: query `AudioSink` (added once playback starts):

```rust
fn pause_music(sinks: Query<&AudioSink, With<Music>>, keys: Res<ButtonInput<KeyCode>>) {
    if keys.just_pressed(KeyCode::KeyM) {
        for sink in &sinks { sink.toggle_playback(); }   // also set_volume, set_speed, stop
    }
}
```

- Global volume: `GlobalVolume` resource.
- Spatial audio: `.with_spatial(true)` + a `SpatialListener` component on
  the camera/player; pan/attenuation follow transforms.
- Formats by feature flag: ogg/vorbis on by default; `mp3`, `flac`, `wav`
  features available. Prefer ogg.

## Animation

Three tiers — pick the lightest that works:

### 1. Code-driven (most game logic)

Mutate `Transform`/fields in systems with `time.delta_secs()`, easing via
`EasingCurve` (`transforms-time.md`). Sprite-sheet animation: timer +
atlas index (`rendering-2d.md`). This covers most 2D needs; `bevy_tweening`
(ecosystem) adds declarative tweens.

### 2. glTF skeletal animation (3D characters)

```rust
#[derive(Resource)]
struct Animations { graph: Handle<AnimationGraph>, idle: AnimationNodeIndex, run: AnimationNodeIndex }

fn setup(mut commands: Commands, assets: Res<AssetServer>,
         mut graphs: ResMut<Assets<AnimationGraph>>) {
    let (graph, indices) = AnimationGraph::from_clips([
        assets.load(GltfAssetLabel::Animation(0).from_asset("character.glb")),
        assets.load(GltfAssetLabel::Animation(1).from_asset("character.glb")),
    ]);
    let graph = graphs.add(graph);
    commands.insert_resource(Animations { graph, idle: indices[0], run: indices[1] });
    commands.spawn(SceneRoot(assets.load(GltfAssetLabel::Scene(0).from_asset("character.glb"))));
}

// The AnimationPlayer lives on a child entity spawned by the scene. Attach the
// graph when players appear:
fn attach(mut commands: Commands, anims: Res<Animations>,
          mut players: Query<(Entity, &mut AnimationPlayer), Added<AnimationPlayer>>) {
    for (e, mut player) in &mut players {
        commands.entity(e).insert(AnimationGraphHandle(anims.graph.clone()));
        player.play(anims.idle).repeat();
    }
}
```

- Control: `player.play(node)`, `.repeat()`, `.set_speed()`, `.pause_all()`,
  `player.stop(node)`; blending = play multiple nodes with weights; smooth
  transitions via `AnimationTransitions::play(&mut player, node, Duration)`.
- The `AnimationGraph` is an asset: clips are nodes, blend nodes combine
  them (`graph.add_blend(...)`).
- Animation events: clips can carry events that fire observers
  (`clip.add_event(...)`, 0.18 `AnimationEventTrigger::target`); useful for
  footstep sounds.
- Masks (`add_target_to_mask_group`) restrict clips to bones subsets
  (upper-body aiming etc.).

### 3. Animating arbitrary properties

`AnimationClip` curves can target any `Reflect`-able property via
`AnimatedField!`/animatable curves — e.g. animate `Sprite.color` or UI
`Node` fields on a timeline. Heavier setup; check
`cargo doc -p bevy_animation` and the `animated_ui` /
`color_animation` examples. For one-off tweens, `bevy_tweening` or manual
easing is less ceremony.

## Gotchas

- No sound on Linux: often missing alsa/pulse dev packages at build time.
- `AnimationPlayer` is on a *descendant* entity of the `SceneRoot`, not the
  root — query `Added<AnimationPlayer>` (as above) or traverse children.
- An `AnimationGraphHandle` must be on the same entity as the player or
  nothing plays.
- wasm audio requires a user gesture before playback (browser policy) —
  start audio after first click/keypress.
