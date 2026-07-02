# Working Offline / Airgapped

How to get exact, version-correct API information without internet, and how
to prepare a machine for airgapped Bevy development.

## Generating API docs locally (the main tool)

```bash
cargo doc -p bevy --no-deps        # fast: just the bevy facade crate
cargo doc                          # everything in the dependency tree (slow once, then cached)
cargo doc --open                   # open in browser (target/doc/)
cargo doc --document-private-items # include non-pub items when reading internals
```

- Docs are generated from the **exact pinned versions** in `Cargo.lock` —
  they can't be stale or wrong-version, unlike memory or tutorials.
- Bevy is a facade over `bevy_ecs`, `bevy_render`, etc.; `-p bevy` docs
  re-export everything reachable. Third-party crates (avian, egui...) are
  covered by the full `cargo doc`.
- Rustdoc search (the `S` key / search box) works fully offline.
- Rust std/book/reference offline: `rustup doc` (`rustup component add rust-docs`).

## Searching source directly (often faster than docs)

All dependency source lives unpacked in the cargo registry:

```bash
ls ~/.cargo/registry/src/*/bevy_ecs-0.19*/
rg "fn get_disjoint_mut" ~/.cargo/registry/src/*/bevy_ecs-0.19*/src/
rg --type rust "pub fn viewport_to_world" ~/.cargo/registry/src/*/bevy_camera*/
```

This answers "does this method exist / what's its exact signature /
what does the doc comment say" definitively. Doc comments in source ≙
rustdoc content.

**Bevy's examples directory is the best learning resource and ships in the
crate**: `~/.cargo/registry/src/*/bevy-0.19*/examples/` — hundreds of
runnable, current examples organized by topic (`2d/`, `3d/`, `ecs/`,
`ui/`, `shader/`, `audio/`, `games/`). When unsure how to wire a feature,
`rg` these examples first.

## Preparing a machine before going offline

During a connected window:

```bash
# 1. Make sure everything resolves & populate caches
cargo fetch                          # downloads all deps in Cargo.lock
cargo build && cargo doc             # warm compile + docs caches

# 2. (Stronger) vendor sources into the repo
cargo vendor vendor/                 # then add to .cargo/config.toml:
#   [source.crates-io]
#   replace-with = "vendored"
#   [source.vendored]
#   directory = "vendor"
```

`cargo vendor` makes the project fully self-contained (committable /
copyable to the airgapped box). Also grab while online:

- Any ecosystem crates you *might* want (add them to `Cargo.toml` —
  even optional/unused — so they vendor; see `ecosystem.md` checklist).
- The Bevy repo itself (`git clone https://github.com/bevyengine/bevy`)
  for examples beyond the published crate, release notes, and migration
  guides (`release-content/` in-repo).
- Rust toolchain components: `rustup component add rust-docs rust-src clippy rustfmt`.
- If targeting wasm: `rustup target add wasm32-unknown-unknown` and tools
  (`wasm-bindgen-cli`, `wasm-server-runner`) installed in advance.

Offline builds then need:

```bash
cargo build --offline      # or [net] offline = true in .cargo/config.toml
```

## Adding a dependency while airgapped

You can only use what's vendored/cached. Check availability:

```bash
ls vendor/ | rg tween                       # vendored?
ls ~/.cargo/registry/cache/*/ | rg avian    # cached .crates?
```

If a needed crate is absent, the options are: implement the slice you need
yourself, or schedule it for the next connected window. Don't fight
`--offline` resolution errors — they always mean "not in cache".

## Answer-finding strategy on an airgapped box (in order)

1. **This skill's reference files** — patterns and idioms.
2. **`rg` the bundled examples** — working code for almost every feature.
3. **`cargo doc` / source `rg`** — exact signatures and doc comments.
4. **Migration file** (`migration.md`) when code/docs disagree — likely a
   version mismatch.
5. **`cargo check` early and often** — the compiler is the final
   offline oracle; let rustc errors steer API discovery (its suggestions
   often name the renamed method).
