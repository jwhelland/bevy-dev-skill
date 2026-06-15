# Ecosystem: Third-Party Crates Worth Knowing

The Bevy ecosystem moves fast and crates pin exact Bevy versions. **Always
check compatibility**: each crate's README has a Bevy-version table;
`cargo add <crate>` + `cargo tree -d` reveals mismatches (a duplicate
`bevy_ecs` in the tree = incompatible versions; symptoms are bizarre trait
errors like "Component is not implemented" for types that clearly are).
On an airgapped machine, whatever is vendored is what you have — check
`cargo tree` before assuming a crate/version is available.

Categories below: the de-facto standard picks first.

## Physics

- **avian** (`avian2d` / `avian3d`) — Bevy-native ECS-driven physics.
  Colliders/rigidbodies as components, queries as the API. Easiest to
  integrate, very popular, supports fixed-timestep + transform
  interpolation. Default choice.
- **bevy_rapier2d / bevy_rapier3d** — wrapper around Rapier; battle-tested,
  more features in some areas (joints, determinism options), less
  ECS-idiomatic.
- Character controllers: **bevy_tnua** (works atop avian or rapier).

## Input

- **leafwing-input-manager** — actions instead of raw keys
  (`Actionlike` enum, `InputMap`), multi-device bindings, rebinding,
  chords. Use for anything beyond hardcoded keys.

## UI

- **bevy_egui** — immediate-mode UI (egui) inside Bevy. Best for tools,
  editors, debug panels; fine for utilitarian game UI.
- **bevy-inspector-egui** — runtime world inspector. Add in dev builds:
  inspect/edit entities, components, resources, assets live.
- **bevy_lunex / sickle_ui / bevy_cobweb_ui** — styled retained-mode UI
  layers over bevy_ui (check maintenance status before adopting).

## Assets & data

- **bevy_asset_loader** — declarative loading states + asset collections
  (`#[derive(AssetCollection)]`), kills handle-bookkeeping boilerplate.
- **bevy_common_assets** — load custom asset types from json/ron/toml/yaml
  with one derive.
- **bevy_embedded_assets** — embed the whole assets folder in the binary
  (single-file distribution; airgap-friendly).
- **bevy_pkv** — simple persistent key-value store (saves/settings), works
  on wasm.

## 2D specifics

- **bevy_ecs_tilemap** — performant chunked tilemaps.
- **bevy_ecs_ldtk** — LDtk level importer (built on bevy_ecs_tilemap).
- **bevy_parallax** — parallax backgrounds.

## Audio

- **bevy_kira_audio** — replaces bevy_audio (disable the `bevy_audio`
  feature) with kira: channels/buses, tweens, precise timing. Use when
  built-in playback (`audio-animation.md`) isn't enough.
- **bevy_seedling** — newer mixer-graph audio (firewheel-based), watch its
  maturity.

## Animation & juice

- **bevy_tweening** — declarative tweens for transforms/colors/any lens.
- **bevy_hanabi** — GPU particle systems (native; check wasm support).
- **bevy_trauma_shake** — screen shake.

## AI / gameplay logic

- **big-brain** — utility-AI (scorers/actions as components).
- **bevy_behave / beet** — behavior trees (newer; check status).
- **seldom_state** — state-machine components.

## Networking

- **bevy_replicon** — server-authoritative replication over the ECS;
  bring-your-own transport (renet integration available).
- **lightyear** — batteries-included client/server netcode with
  prediction/rollback.
- **bevy_matchbox** — WebRTC p2p (incl. wasm) — pairs with **bevy_ggrs**
  for rollback netplay.

## Dev tools & debugging

- **bevy_mod_debugdump** — schedule/render-graph visualization (graphviz).
- **bevy-inspector-egui** (above).
- **bevy_screen_diags / iyes_perf_ui** — on-screen perf panels.

## Misc utilities

- **bevy_cursor / bevy_framepace** (frame pacing/latency),
- **bevy_mod_outline** (mesh outlines), **bevy_atmosphere** (older sky
  alternative to built-in `Atmosphere`), **bevy_water**, **bevy_terrain**
  variants — search crates.io / bevy-assets (bevy.org/assets) when online.
- **moonshine-save** — structured save/load framework.

## Picking a crate: checklist

1. Does its pinned Bevy version match yours? (Hard requirement.)
2. Maintained within the last release cycle? (Bevy minor releases break
   everything; abandoned = stuck on old Bevy.)
3. Could required components / observers / built-ins now cover it? Bevy
   absorbs ecosystem features every release (picking, required components,
   atmosphere were all crates once) — check the built-in option first.
4. On airgapped systems: is it already vendored? If not, plan the
   `cargo vendor` update during a connected window (`offline-docs.md`).
