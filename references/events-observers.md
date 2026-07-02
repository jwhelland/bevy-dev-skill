# Messages, Events, and Observers

Targets Bevy 0.19. Since 0.17 Bevy has **two distinct systems** that were
both called "events" before:

| | Messages (buffered) | Events (observed) |
|---|---|---|
| Trait | `Message` | `Event` / `EntityEvent` |
| Send | `MessageWriter`, `world.write_message` | `commands.trigger`, `world.trigger` |
| Receive | `MessageReader` (polled by systems) | Observers (run immediately) |
| Timing | next time reader system runs | synchronously at trigger / command apply |
| Targeting | none | optional per-entity |
| Use for | high-volume streams, frame-batched processing | reactions, callbacks, entity lifecycle |

Code written for ≤0.16 used `Event`/`EventReader`/`EventWriter`/`add_event`
for the buffered kind — that is `Message`/`MessageReader`/`MessageWriter`/
`add_message` now (see `migration.md`).

## Messages (buffered)

```rust
#[derive(Message)]
struct DamageDealt { target: Entity, amount: f32 }

app.add_message::<DamageDealt>();

fn attack(mut writer: MessageWriter<DamageDealt>) {
    writer.write(DamageDealt { target, amount: 10.0 });
}

fn apply_damage(mut reader: MessageReader<DamageDealt>, mut q: Query<&mut Health>) {
    for msg in reader.read() {
        if let Ok(mut hp) = q.get_mut(msg.target) { hp.current -= msg.amount; }
    }
}
```

- Each `MessageReader` tracks its own cursor; multiple readers each see
  every message.
- Messages live for **two frames** (double-buffered). A reader system gated
  off by a run condition for longer than that silently drops messages —
  either keep the reader running or manage `Messages<M>` manually with
  custom clearing.
- Writers/readers of different message types parallelize; two writers of
  the same type conflict.
- Order systems `writer.before(reader)` for same-frame handling, otherwise
  delivery slips one frame (usually fine).

## Events and observers

```rust
#[derive(Event)]
struct WaveFinished { wave: u32 }

// Global observer: runs whenever the event is triggered, immediately.
app.add_observer(|ev: On<WaveFinished>, mut next: ResMut<NextState<GameState>>| {
    info!("wave {} done", ev.wave);   // On derefs to the event
});

fn somewhere(mut commands: Commands) {
    commands.trigger(WaveFinished { wave: 3 });   // runs observers at command apply
}
// world.trigger(...) runs them synchronously, right there.
```

`On<E>` (named `Trigger<E>` in 0.16) gives: `ev.event()` (or deref),
`ev.event_mut()`, `ev.observer()`. Observers are full systems — any system
params after the `On` parameter.

### EntityEvent — targeted at one entity

```rust
#[derive(EntityEvent)]
struct Explode { entity: Entity }     // field named `entity`, or mark with #[event_target]

// Per-entity observer:
commands.entity(bomb).observe(|ev: On<Explode>, mut commands: Commands| {
    commands.entity(ev.entity).despawn();
});

// Global observers of Explode ALSO run for every target.
commands.trigger(Explode { entity: bomb });
```

- `commands.entity(e).observe(...)` attaches an observer to one entity —
  this is the idiomatic "callback on this button/unit" mechanism.
- Observers are themselves entities (`Observer` component); despawning the
  watched entity cleans up its observers.

### Propagation (bubbling)

```rust
#[derive(EntityEvent)]
#[entity_event(propagate)]      // default propagation = up ChildOf hierarchy
struct Clicked { entity: Entity }
```

In an observer, `ev.propagate(false)` stops bubbling;
`ev.original_event_target()` gives the entity the event started at. UI
picking events (`On<Pointer<Click>>` etc.) use this — see `ui.md`.

### Lifecycle events

Observers can watch component add/remove without polling:

```rust
app.add_observer(|ev: On<Add, Enemy>, mut commands: Commands| {
    // runs immediately when Enemy is added to any entity; ev.entity is the target
});
```

Available: `Add`, `Insert`, `Replace`, `Remove`, `Despawn` (pre-0.17 names:
`OnAdd`, `OnInsert`, ...). Semantics: `Add` = newly present; `Insert` =
every insert including overwrites; `Replace`/`Remove` = before value goes
away; `Despawn` = entity being despawned. Compare with `Added<T>`/
`RemovedComponents<T>` which are *polled* (batched, cheaper, but delayed to
the reading system's next run).

## Choosing between the mechanisms

- **Stream of many similar things per frame** (damage numbers, collision
  pairs, network packets): Message. Cheap, batch-processed, parallel.
- **Reaction to a discrete occurrence**, especially entity-specific
  (clicked, died, equipped): Event + observer. Runs immediately, can
  cascade (observers may trigger further events — depth-first).
- **Enforcing component invariants**: component hooks (`ecs-advanced.md`),
  which cannot be skipped.
- **Decoupling plugins**: either works; messages keep timing predictable,
  observers keep latency zero. Don't build long synchronous observer chains
  for core game flow — they're hard to reason about and order; prefer
  messages + explicitly ordered systems for the main loop.

## Gotchas

- Observer ordering among multiple observers of the same event is
  unspecified — don't depend on it.
- An observer triggering the same event type it observes = recursion;
  guard or redesign.
- `commands.trigger` is deferred like all commands; `world.trigger` is not.
- Messages need `add_message` registration; events need none (observers
  self-register).
