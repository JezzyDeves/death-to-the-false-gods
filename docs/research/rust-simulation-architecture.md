# Research: Rust simulation architecture inside Tauri

Resolves issue #4. Researched 2026-11-17 against the sources listed at the end.

## Question

What are the sound options for running a deterministic game simulation in the Rust half of a Tauri 2 app and showing it in a SvelteKit SPA? Five areas: state ownership and IPC, simulation structure, determinism, data files for authored content, and save and load.

Scope it is sized to (from the map, issue #1): turn-based one-on-one combat on deeply nested anatomical bodies, creatures authored as data files with no code change, same seed gives the same world, one save slot with no migration guarantees, one developer.

## How to read this document

Every factual claim carries one of three markers:

- **[S1]**, **[S2]**, ...: verified. I read the cited source during this research. The source list at the end gives the URL and the version read.
- **[memory]**: recalled, not checked against a source in this research. Treat as unverified.
- **[assessment]**: my own judgement or design suggestion, derived from the verified facts and the alpha's scope. Not a sourced fact.

## Answer in brief

1. **State and IPC.** Rust owns all game state in one `Mutex<Game>` registered as Tauri managed state. The UI sends player intent through commands and renders the view model each command returns. Events and channels are not needed for the turn loop.
2. **Simulation structure.** Plain structs and arenas (`Vec` plus typed index newtypes), no ECS. Put the simulation in its own library crate with no Tauri dependency.
3. **Determinism.** `rand` 0.10 with a named portable generator (`Xoshiro256PlusPlus`, built into `rand`, or `rand_pcg::Pcg64`), never `StdRng` or `SmallRng`. Integer arithmetic in the simulation, ordered collections only, single-threaded, a golden-seed test, and a committed `Cargo.lock`.
4. **Data files.** RON 0.12 for body plans, strict parsing with `deny_unknown_fields`, then a separate validation pass that reports every error. Keep content outside `src-tauri/`. Reload through an explicit command first; add a `notify` watcher later if wanted.
5. **Save and load.** One whole-state serde snapshot written as JSON with `serde_json` (already a dependency), including the RNG state and a format version, written to a temporary file and renamed into place. Do not use `bincode`.

## Baseline: what the repo has today

- `src-tauri/Cargo.toml` asks for `tauri = "2"`, `tauri-build = "2"`, `tauri-plugin-opener = "2"`, `serde = "1"` with `derive`, `serde_json = "1"`, edition 2024. [S1]
- No `src-tauri/Cargo.lock` is committed, so the resolved Rust versions are not pinned. The newest `tauri` on crates.io is 2.12.6. [S1][S2]
- `bun.lock` pins `@tauri-apps/api` and `@tauri-apps/cli` at 2.12.5. [S1]
- The local toolchain is rustc 1.99.1, which is newer than the minimum Rust version of every crate discussed here (the highest is `bevy_ecs` 0.20.1 at 1.95). [S1][S10]

Current versions of the candidate crates, from the crates.io API on 2026-11-17 [S2]:

| Crate                   | Newest stable     | Note                                                          |
| ----------------------- | ----------------- | ------------------------------------------------------------- |
| `tauri`                 | 2.12.6            |                                                               |
| `bevy_ecs`              | 0.20.1            |                                                               |
| `hecs`                  | 0.11.2            |                                                               |
| `slotmap`               | 1.1.1             |                                                               |
| `rand`                  | 0.10.4            |                                                               |
| `rand_chacha`           | 0.10.0            | The Rand book says it is likely to be discontinued [S14]      |
| `rand_pcg`              | 0.10.2            |                                                               |
| `rand_xoshiro`          | 0.8.1             |                                                               |
| `ron`                   | 0.12.3            |                                                               |
| `toml`                  | 1.2.1             | Implements TOML spec 1.1.0                                    |
| `serde_json`            | 1.0.154           |                                                               |
| `serde_yaml`            | 0.9.34+deprecated | Unmaintained [S27]                                            |
| `serde-saphyr`          | 2.1.1             | YAML deserializer, actively released                          |
| `serde_path_to_error`   | 0.1.20            |                                                               |
| `notify`                | 8.2.2             | 9.0.0-rc.9 is in release-candidate stage                      |
| `notify-debouncer-full` | 0.7.0             | 0.8.0-rc.4 is in release-candidate stage                      |
| `postcard`              | 1.1.3             | Last published 2025-07-24                                     |
| `bincode`               | 3.0.0             | A stub that fails to compile; the crate is unmaintained [S31] |
| `rmp-serde`             | 1.3.1             |                                                               |
| `ciborium`              | 0.2.2             | Last published 2024-01-24                                     |
| `indexmap`              | 2.14.2            |                                                               |
| `tauri-specta`          | 1.0.2 (Tauri 1)   | The Tauri 2 line is still 2.0.0-rc.29 [S9]                    |
| `ts-rs`                 | 12.0.1            |                                                               |

---

## 1. State ownership and IPC

### What Tauri 2 provides

**Managed state.**

- `app.manage(value)` registers one value per type; commands receive it as a `State<'_, T>` parameter, and anything with an `AppHandle` can fetch it with `app.state::<T>()`. [S3]
- Mutation needs interior mutability. The docs use `std::sync::Mutex` and say the standard mutex is usually the right choice; an async mutex is only needed when the guard must be held across an `.await`. [S3]
- `Arc` is unnecessary; Tauri wraps managed state itself. [S3]
- Asking for a `State` type that was never managed is not a compile error. The `invoke` call is rejected at runtime, and `Manager::state()` panics. The docs suggest a type alias (`type AppState = Mutex<AppStateInner>`) to avoid the mistake. [S3]
- `manage()` returns `false` if a value of that type is already managed. [S7]

**Commands** (UI calls Rust, gets a reply).

- A `#[tauri::command]` function takes any `serde::Deserialize` arguments and returns any `serde::Serialize` value; the JS `invoke()` returns a promise. Return values are serialized to JSON. [S4]
- Returning `Result<T, E>` rejects the promise on `Err`. `E` must implement `Serialize`; the docs show a `thiserror` enum with a manual `Serialize` impl, optionally tagged so the UI gets `{ kind, message }`. [S4]
- Commands without `async` run on the main thread. `async` commands run on a separate task through `async_runtime::spawn`. `#[tauri::command(async)]` moves a non-async function off the main thread. [S4]
- An `async` command with a borrowed argument such as `State<'_, T>` must return a `Result`. [S3][S4]
- Every command must be listed in a single `tauri::generate_handler![...]` call; a second `invoke_handler` call replaces the first. Command names are global, not module-scoped. Commands in `lib.rs` cannot be `pub`; commands in other modules must be. [S4]
- Arguments are camelCase in JS by default; `#[tauri::command(rename_all = "snake_case")]` changes that. [S4]

**Events** (either side emits, any number of listeners).

- `app.emit(name, payload)` sends to all listeners; `emit_to` and `emit_filter` target specific webviews. The payload must be `Serialize + Clone`. [S5]
- The docs position events for "small amounts of data" and multi-producer, multi-consumer patterns, and say they are "not designed for low latency or high throughput". Events are not type safe, payloads are always JSON strings, they cannot return a value, and they are implemented by evaluating JavaScript in the webview. [S4][S5]

**Channels** (Rust streams ordered messages to one caller).

- The UI creates a `Channel<T>`, passes it as a command argument, and sets `onmessage`; Rust calls `channel.send(value)`. [S5]
- The docs call channels "designed to be fast and deliver ordered data" and the recommended mechanism for streaming; Tauri uses them internally for download progress, child process output, and WebSocket messages. [S4][S5]
- In 2.12.6 the Rust type is `tauri::ipc::Channel<TSend>` and `send` returns `tauri::Result<()>`. [S7]

**Testing.** `tauri::test` offers a mock runtime (`mock_builder`, `get_ipc_response`), but the module documents itself as unstable. [S8]

### Options

| Option                                                                                           | For                                                                                                                                   | Against                                                                                                                                        |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. Commands only.** UI invokes an intent command; Rust mutates state and returns a view model. | One round trip per player action matches a turn-based loop exactly. Typed arguments and results. Errors arrive as promise rejections. | Rust cannot push anything unprompted.                                                                                                          |
| **B. Commands in, events out.** Commands return nothing; Rust emits state-changed events.        | Rust can push at any time; several windows can listen.                                                                                | Untyped JSON-string payloads, no reply, and the UI must reconcile event order with its own requests. Solves a problem the alpha does not have. |
| **C. Commands plus a channel.** A long-running command streams progress through a `Channel`.     | Ordered, typed streaming for a slow operation such as world generation.                                                               | Extra moving part; only worth it once something is slow enough to need a progress display.                                                     |

All assessments in the table are **[assessment]**, built on the verified behaviour above.

### Recommendation

Use **A** for the whole alpha. **[assessment]**

- One managed value, `Mutex<Game>` behind a type alias, using `std::sync::Mutex`. A turn never awaits, so no async mutex is needed.
- Commands are the only write path. Each takes a player intent (`start_new_game { seed }`, `choose_travel { .. }`, `take_combat_action { .. }`, `save_game`, `load_game`) and returns a serializable **view model**: the prose lines produced by that step, the choices now available, and the body-status panel data. The UI stores only the latest view model in `$state` and never computes game rules.
- Add one read-only `current_view` command so the UI can re-fetch after a reload of the webview.
- Keep each command a few lines: lock, call into the simulation crate, map the result to a view model, return. All logic lives in the simulation crate (section 2), so it is testable with plain `cargo test` and the unstable `tauri::test` module is not needed.
- Use a non-async command for a turn; it is far too short to freeze the UI. Mark world generation `async` (returning `Result`, because it borrows `State`) so the main thread stays free.
- Return `Result<View, GameError>` with a serializable error enum so content errors and save errors reach the UI as structured rejections.
- Reach for a **channel** only if world generation becomes slow enough to want progress lines, and for an **event** only for the development-time "content reloaded" notice in section 4, which is exactly the small, unprompted, low-frequency message events are meant for.

**TypeScript types for the view model.** The view-model surface is small. Hand-written TS types mirroring the serde structs are enough to start. `tauri-specta` would generate them, but its Tauri 2 line is still a release candidate (2.0.0-rc.29) [S2][S9]; `ts-rs` 12.0.1 is a stable derive-based alternative [S2], though I did not read its documentation. **[assessment]**

---

## 2. Simulation structure

### Options

**A. `bevy_ecs` 0.20.1 used standalone.**

- The README states it "can easily be used as a standalone crate in other projects". [S10]
- It brings systems, schedules, change detection, messages, observers, and a built-in parent-child relationship (`ChildOf` / `Children`) that despawns descendants with the parent. [S10][S11]
- Default features pull in `bevy_reflect` and an async executor, and the crate depends on `bevy_tasks`, `bevy_utils`, `bevy_platform`, `bevy_ptr`, and others. [S10]
- Query iteration order is documented as "not guaranteed". [S11]
- `Entity` can derive serde traits under the `serialize` feature, but its docs warn that this makes "zero guarantee of long term wire format compatibility". [S11] I found no whole-`World` serde support inside `bevy_ecs` itself; in Bevy that job belongs to the scene and reflection crates. **[memory]**
- The schedule's parallel executor exists; the `multi_threaded` feature is not in the standalone crate's default features. [S10]
- Bevy ships breaking releases several times a year, each with a migration guide. **[memory]**

**B. `hecs` 0.11.2.**

- Describes itself as "a library, not a framework": a `World` you query from ordinary code, no system abstraction. [S12]
- Its own README has a "Why Not ECS?" section: "If your game will have few types of entities, consider a simpler architecture such as storing each type of entity in a separate plain `Vec`", and "ECS may be overkill for games that don't call for batch processing of entities." [S12]
- Entities are "yielded in arbitrary order", and `World` is documented as "an unordered collection of entities". [S13]
- Serialization exists (`row-serialize`, `column-serialize` features) but requires a hand-written `SerializeContext` that lists every component type. [S13]
- No built-in hierarchy; parent links would be ordinary components. **[memory]**
- Version 0.11.0 made breaking changes to query iteration, and 0.11.1 and 0.11.2 fixed several undefined-behaviour bugs. [S12]

**C. Plain structs and arenas.**

- A `Body` owns a flat `Vec<Part>`; a part refers to its parent and children by a typed index newtype (`PartId(u32)`), and holds its tissue layers and organs. A `World` struct owns a handful of such tables. **[assessment]**
- `Vec` iteration order is the insertion order, so ordering is deterministic by construction. `#[derive(Serialize, Deserialize)]` on the whole tree gives save and load for free. **[assessment]**
- Where entries are removed and stale handles could be reused, `slotmap` 1.1.1 gives versioned keys, so a removed key stays invalid when its slot is reused. With its `serde` feature, keys and maps serialize, and "a key remains valid for a slot map even after one or both have been serialized and deserialized". `new_key_type!` gives each table a distinct key type. [S15]

### Trade-offs

**[assessment]** throughout this subsection.

- **What an ECS buys.** Loose composition across many entity kinds and fast batch iteration over thousands of entities. The alpha has two bodies in a fight and a few settlements; neither benefit applies.
- **What an ECS costs here.** Bodies are trees with layers inside each node. In an ECS every part and tissue becomes an entity joined by relationship components, so "what did this blade pass through" becomes a walk across entity lookups instead of a walk over a struct. Iteration order is unspecified in both crates, so every query whose order matters must be sorted to keep determinism. Saving needs extra machinery in both. `bevy_ecs` adds a large dependency tree and an upgrade treadmill.
- **What plain structs cost.** Adding a cross-cutting behaviour later means touching the structs that carry it. For one developer on one codebase that is an ordinary refactor.
- **Composable anatomy does not need an ECS.** "Creatures built from reusable parts" is a data-authoring property: body-plan files reference part templates, and the loader assembles them into a `Body` tree. That is independent of how the runtime stores entities.

### Recommendation

Use **C**. **[assessment]**

- Start with `Vec` plus index newtypes for body parts. Parts are created once when a body is instantiated; a severed limb can be flagged rather than removed, so plain indices never dangle.
- Use `slotmap` (with `serde`) only for world-level tables where things are genuinely removed and handles are held elsewhere (creatures, items). Skip it until that need appears.
- Put the simulation in its own library crate with no `tauri` dependency (for example a `sim` crate in a Cargo workspace next to `src-tauri`). The Tauri crate depends on it and only adapts it to commands. `tauri dev` already watches "dependent crates in the workspace" and rebuilds on change. [S6]
- Shape the core as one pure-looking step: `fn step(&mut self, action) -> Vec<Event>`, with the RNG stored inside the state. The event list feeds the prose templates and doubles as the test oracle.
- Revisit an ECS only if the game later grows to many simultaneously simulated actors with overlapping behaviours. `hecs` would be the smaller step from this design.

---

## 3. Determinism

### RNG crates and what they promise

The `rust-random` project's reproducibility policy [S14]:

- `rand::rngs::StdRng` and `rand::rngs::SmallRng` are declared **non-portable**: deterministic, but allowed to give different results on different platforms and in any release. From 0.10 that includes patch releases.
- Everything else in the public API (named PRNGs, distributions, sequence algorithms) is **portable**: reproducible across platforms and across patch releases. Minor releases may make value-breaking changes, which must be noted in the changelog.
- "Seeding does not imply reproducibility of results. For that you need to use a named RNG with a fixed algorithm (e.g. `ChaCha12Rng` not `StdRng`)."
- Never sample `usize` or `isize` directly if portability matters. `random_range` on `usize` uses `u32` sampling where it can.
- Float results from distributions may not be portable, because transcendental functions differ by platform.
- The 0.10 update guide reports "no known value-breaking changes to `rand` in v0.10".

Candidate generators:

| Generator                  | Where                           | Portable                                    | `Clone`           | serde                                    | Notes                                                                                                                                                                           |
| -------------------------- | ------------------------------- | ------------------------------------------- | ----------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `StdRng`, `SmallRng`       | `rand` 0.10.4                   | No [S14][S16]                               | No (StdRng) [S16] | No                                       | Ruled out by policy.                                                                                                                                                            |
| `Xoshiro256PlusPlus`       | `rand` 0.10.4, no extra feature | Yes [S16]                                   | Yes [S16]         | Yes, with `rand`'s `serde` feature [S16] | 32 bytes of state. No extra crate.                                                                                                                                              |
| `Pcg64`, `Pcg32`           | `rand_pcg` 0.10.2               | Yes, tested against reference vectors [S17] | Yes [S17]         | Yes, with its `serde` feature [S17]      | Has a stream parameter, useful for independent sub-streams.                                                                                                                     |
| `ChaCha8Rng` / `12` / `20` | `rand` 0.10.4, feature `chacha` | Yes [S16]                                   | No [S14]          | No [S14]                                 | Deliberately not cloneable or serializable in 0.10. State can be saved by hand through `serialize_state()` (49 bytes) or `get_seed` / `get_stream` / `get_word_pos`. [S14][S18] |
| `ChaCha8Rng` / `12` / `20` | `rand_chacha` 0.10.0            | Yes **[memory]**                            | Yes [S18]         | Yes, with its `serde` feature [S18]      | "Likely to be discontinued in the future" per the Rand book. [S14]                                                                                                              |

Other 0.10 facts that affect code written against older tutorials: the user-facing extension trait is now `rand::RngExt` (the core trait is `Rng`), `SeedableRng::seed_from_u64` exists for integer seeds, and the `unbiased` cargo feature "is expected to affect reproducibility of results". [S14][S16]

### Pitfalls that break same-seed-same-world

- **Hash map iteration order.** `std::collections::HashMap` is "randomly seeded" by default, and its iterators visit entries "in arbitrary order". [S19] `BTreeMap` iterates in key order [S19]; `IndexMap` iterates in insertion order independent of the hash, except after `remove` / `swap_remove`. [S20]
- **ECS query order**, if an ECS were used: not guaranteed in `bevy_ecs`, arbitrary in `hecs`. [S11][S13]
- **Floats.** `+ - * /` are specified as correctly rounded IEEE 754 operations, so they are reproducible. Functions such as `powf`, `powi`, `sin`, `exp`, and `ln` have "unspecified precision" that "varies by platform, Rust version, and can even differ within the same execution from one invocation to the next". NaN bit patterns are not guaranteed portable or deterministic. [S19]
- **Float round-trips through a save.** `serde_json` only guarantees `f64 -> JSON -> f64` identity with its `float_roundtrip` feature, and it writes NaN and infinity as `null`. [S24]
- **Directory listing order.** `std::fs::read_dir` order "is platform and filesystem dependent" and "can change between calls". [S19] If content IDs or RNG draws depend on the order files were loaded, the world depends on the filesystem.
- **Threads.** Async Tauri commands run on other threads [S4]. Two commands racing for the mutex would apply in an unspecified order. Any parallel iteration inside the simulation would do the same to RNG draws. **[assessment]**
- **Ambient inputs.** Wall-clock time, `rand::rng()` (OS-seeded), and anything derived from memory addresses are different every run. **[assessment]**
- **Version drift.** A minor `rand` release may change values [S14]; without a committed `Cargo.lock`, a fresh checkout can resolve different versions. Tauri's docs say the lockfile should be committed. [S6]
- **Content is an input.** The same seed with an edited body plan is a different world. That is expected, but it means "same seed" really means "same seed, same content, same build". **[assessment]**

### Recommendation

**[assessment]**, resting on the verified facts above.

- Depend on `rand = { version = "0.10", features = ["serde"] }` and use **`rand::rngs::Xoshiro256PlusPlus`** seeded with `seed_from_u64(world_seed)`. It is a named portable generator, needs no second crate, and is `Clone` and serializable, which section 5 needs. `rand_pcg::Pcg64` is an equally good choice if explicit streams are wanted. Do not use the ChaCha types: cryptographic strength is irrelevant here, and in 0.10 they cannot be cloned or serialized directly.
- Store the generator inside the game state and pass `&mut` to whatever needs randomness. No global RNG, no `rand::rng()` anywhere in the simulation crate.
- Give independent subsystems their own generator, derived once from the world seed in a fixed order (world generation, then each encounter). Adding a roll to combat then cannot shift world generation.
- Use integers (fixed-point where fractions are needed) for every quantity that feeds a simulation decision: tissue thickness, force, blood volume, probabilities as parts per thousand. Keep floats for presentation. This sidesteps transcendental-function drift and the JSON round-trip issue, and makes test snapshots exact.
- Sample `u32` or `u64`, not `usize`.
- In simulation state use `Vec`, `BTreeMap`, or `IndexMap`. Do not iterate a `HashMap` or `HashSet`. Clippy can enforce this with a `disallowed-types` entry in `clippy.toml`. **[memory]**
- Sort directory listings before loading content, and key content by its authored string ID, never by load position.
- Keep the simulation single-threaded behind the one mutex; have the UI await each `invoke` before sending the next.
- Commit `src-tauri/Cargo.lock`.
- Add one golden test early: a fixed seed and a fixed list of actions must produce a known event log (or its hash). It catches every pitfall above, including a value-breaking dependency upgrade. This also answers part of the map's open "testing the simulation" question.

---

## 4. Data files for authored content

### Formats

**RON (`ron` 0.12.3).**

- Syntax mirrors Rust: structs `( field: value )`, enum variants with payloads (`Unit`, `Newtype(1.5)`, `Struct(x: 1.0)`), `Some` / `None`, tuples, lists, maps, comments, and trailing commas. Designed to cover the whole serde data model. [S21]
- Errors are a `SpannedError { code, span }` whose `Display` prints `line:col: message` (or a `start-end` range). [S21]
- Limitations: not fully self-describing, so `#[serde(tag = ...)]`, `#[serde(untagged)]`, and `#[serde(flatten)]` have only limited support, with a long list of restrictions. Plain externally tagged enums, serde's default, are the native case. [S21]
- Still pre-1.0. The 0.12.3 changelog is mostly bug fixes plus small format additions. [S21]
- Editor plugins exist for VS Code, IntelliJ, Zed, Vim, and others, plus a third-party LSP. [S21]
- Has a configurable recursion limit. [S21]

**TOML (`toml` 1.2.1).**

- Errors carry a byte span and render a source excerpt with a caret under the offending text ("TOML parse error at line 1, column 10" followed by the line). [S22]
- Ships `serde_spanned` for capturing the span of any value, which lets a later validation pass point at the exact place. [S22]
- Stable 1.x crate. [S2]
- Deeply nested arrays of tables are written as repeated `[[a.b.c]]` headers or inline tables, which becomes hard to read for trees several levels deep. **[memory]** and **[assessment]**

**JSON (`serde_json` 1.0.154).**

- Already a dependency. [S1]
- Errors expose `line()`, `column()`, and a category (syntax, data, EOF, I/O). [S23]
- Trailing commas are a syntax error [S23]; the format has no comments **[memory]**. Enum variants with data are written as single-key objects **[memory]**. Default nesting limit of 128 levels. [S24]

**YAML.**

- `serde_yaml` is marked "no longer maintained" by its author. [S27]
- `serde-saphyr` 2.1.1 is an actively released YAML deserializer. [S2][S28]
- Indentation-sensitive syntax and implicit scalar typing make hand-authored nested data error-prone. **[memory]**

### Validation and error messages

- `#[serde(deny_unknown_fields)]` makes an unknown field an error. Without it, self-describing formats silently ignore unknown fields, so a misspelled optional field would vanish. It cannot be combined with `flatten`. [S25]
- `serde_path_to_error` 0.1.20 wraps any serde `Deserializer` and reports the field path to a failure, for example `dependencies.serde.version`. [S26] I did not test it against the `ron` deserializer specifically.
- serde only checks shape. Rules such as "every referenced part template exists", "exactly one root part", "no cycles", and "an organ sits inside a part that can contain it" need a pass of their own. **[assessment]**

### Reload during development

- `tauri dev` "watches your `src-tauri` folder and its dependent crates in the workspace for changes, so your application is automatically rebuilt and restarted". `.taurignore` files exclude paths, and `--no-watch` turns it off. [S6] So a content file saved inside `src-tauri/` restarts the app instead of reloading in place.
- `notify` 8.2.2 is the cross-platform watcher (FSEvents on macOS, inotify on Linux, ReadDirectoryChangesW on Windows, polling everywhere). Its 8.2.2 docs warn that editors save differently (truncate and write, or write a new file and replace), so precise event kinds are unreliable. [S29]
- `notify-debouncer-full` 0.7.0 sits on top of it, merges bursts of events over a timeout, and delivers batches to a callback or channel. [S29]
- `tauri-plugin-fs` 2.7.1 has a `watch` feature built on the same two crates [S30]. It exposes watching to the frontend **[memory]**, which would move file handling into the UI. **[assessment]**
- For shipping, `bundle.resources` in `tauri.conf.json` copies files or folders into the app bundle, and the path resolver finds them at runtime. [S6]

### Options

| Option   | For                                                                                                                                 | Against                                                                                       |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **RON**  | Enum variants with data, optional values, and nesting read like the Rust types they load into. Comments. Line and column in errors. | Pre-1.0. Smaller tooling ecosystem. Avoid `tag`, `untagged`, and `flatten` on content types.  |
| **TOML** | Best built-in error rendering. Stable. Familiar.                                                                                    | Poor fit for deep trees of parts, layers, and organs.                                         |
| **JSON** | No new dependency. Universal tooling.                                                                                               | No comments, no trailing commas, verbose enums. Unpleasant to author by hand at this depth.   |
| **YAML** | Compact for nested data.                                                                                                            | The long-standing crate is unmaintained; the format's typing and indentation pitfalls remain. |

Fit judgements are **[assessment]**.

### Recommendation

**[assessment]**

- **Format: RON.** Body plans are trees of tagged alternatives (tissue kinds, organ functions, joint types), which is exactly what RON writes most naturally. Keep content enums externally tagged (serde's default) and do not use `flatten`, which also keeps `deny_unknown_fields` available.
- **Two layers of types.** `*Def` structs mirror the files and derive `Deserialize` with `deny_unknown_fields`. A `compile` step turns them into the runtime types the simulation uses, resolving string IDs to indices. The runtime never sees an unvalidated plan.
- **Composition by reference.** Part templates live in their own files and are referenced by string ID from body plans, so a new creature is a new file.
- **Report every problem, with location.** Parse errors: file path plus RON's `line:col` message. Validation errors: file path plus the logical path (`wolf.parts["torso"].organs[2]`) plus the rule broken. Collect all validation errors in one pass rather than stopping at the first.
- **Test the shipped content.** One `cargo test` that loads and validates every file under the content directory, so a broken plan fails the build, not the fight.
- **Location.** A top-level `content/` directory outside `src-tauri/`, so that saving a file does not trigger the `tauri dev` rebuild. Resolve it from `CARGO_MANIFEST_DIR` in debug builds. For a packaged build, list it under `bundle.resources`; the alpha may never need that step.
- **Reload.** Start with an explicit `reload_content` command bound to a development-only button or key: re-read, re-validate, and on success swap the content and restart the current scenario from its seed (determinism makes that replay exact). On failure keep the old content and show the errors. Add a `notify-debouncer-full` watcher behind `#[cfg(debug_assertions)]` that calls the same function and emits a `content-reloaded` or `content-error` event only if pressing the key becomes tedious. Watch the directory, not individual files, and rely on the debouncer rather than on event kinds.

---

## 5. Save and load

### Options

| Option                                                       | For                                                                                                                                        | Against                                                                                                                                                                                                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **JSON snapshot (`serde_json`)**                             | Already a dependency [S1]. Human-readable, so a save can be inspected and diffed when a bug appears. Line and column on load errors [S23]. | Map keys must be strings or numbers, which are written as strings; other key types are an error [S24]. NaN and infinity become `null` [S24]. Exact `f64` round-trip needs `float_roundtrip` [S24]. 128-level nesting limit [S24]. Largest files. |
| **RON snapshot (`ron`)**                                     | Same crate as content. Readable. Handles enums and non-string map keys natively [S21].                                                     | Pre-1.0 format for persisted data. Slower and less tooling than JSON **[memory]**.                                                                                                                                                               |
| **`postcard` 1.1.3**                                         | Compact binary with a documented, stable wire format since 1.0 [S32].                                                                      | Not self-describing: `deserialize_any` is refused outright [S32], so untagged and internally tagged enums and `flatten` cannot load. Opaque when debugging. Any struct change makes old files unreadable **[assessment]**.                       |
| **`bincode`**                                                | None.                                                                                                                                      | Unmaintained. Version 3.0.0 on crates.io is a stub whose `lib.rs` is a single `compile_error!` [S31].                                                                                                                                            |
| **`rmp-serde` 1.3.1 (MessagePack), `ciborium` 0.2.2 (CBOR)** | Self-describing binary; smaller than JSON.                                                                                                 | No benefit at this size. `ciborium`'s last release was January 2024 [S2].                                                                                                                                                                        |
| **Seed plus input log, replayed on load**                    | Tiny file. Reuses the determinism work. The same log is the golden test from section 3.                                                    | Any rules or content change invalidates the save. Load time grows with play time. **[assessment]**                                                                                                                                               |
| **ECS world serialization**                                  | Only relevant if an ECS is chosen.                                                                                                         | Hand-written per-component context in `hecs` [S13]; reflection and scene machinery in Bevy **[memory]**.                                                                                                                                         |

Supporting facts:

- The plain-struct design from section 2 makes the entire game state one `#[derive(Serialize, Deserialize)]` tree. **[assessment]**
- `Xoshiro256PlusPlus` and the PCG generators serialize under their `serde` features. [S16][S17]
- `slotmap` keys and maps survive a serde round trip with keys still valid. [S15]
- `std::fs::rename` replaces an existing destination file and does not work across mount points. [S19]
- `app.path().app_data_dir()` returns the per-app data directory, derived from the bundle identifier. [S7]

### Recommendation

**[assessment]**

- **One whole-state snapshot, written as JSON with `serde_json`.** The state is small, the single developer is also the only person who will ever debug a save, and readability is worth more than size. The JSON limits do not bite under the other recommendations: integers instead of floats, `Vec` and string-keyed maps in state, and a flat part arena instead of recursive nesting.
- **Envelope.** `SaveFile { format_version, seed, rng, game }`. On load, a `format_version` mismatch is a clear error ("this save is from an older build"), which is all that "no migration guarantees" requires.
- **Save the RNG state, not only the seed,** so that play after loading matches play that was never interrupted.
- **Save instantiated bodies in full.** A wounded body is a mutated copy of its body plan. Storing the whole body means a save does not silently change when a content file is edited. Store the content ID alongside for display and debugging.
- **Write safely.** Serialize to `save.json.tmp` in the app data directory, then rename it over `save.json`.
- **When to save** is a design question for the map's "Save and load" item, not an architecture one. A snapshot is cheap enough to write at every safe point (after each travel step and after each fight), which also fits permadeath: on death, delete the file.
- **Do not** adopt `bincode`. Consider `postcard` only if a save ever becomes large enough to matter.
- Keep the seed-plus-input-log idea for tests rather than for saves.

---

## Could not verify

These were not checked against a primary source in this research:

- That `bevy_ecs` has no whole-`World` serde support of its own, and that Bevy's release cadence is several breaking releases a year. I verified only the `Entity` serialization warning and the feature list.
- That `hecs` has no built-in hierarchy support.
- That `rand_chacha` 0.10.0 falls under the same portability policy as the other named generators. I verified its `Clone` and serde support and its discontinuation notice, not a portability statement in its own docs. Its source is no longer in the `rand` repository's default branch.
- Whether `serde_path_to_error` composes cleanly with the `ron` deserializer. Its README demonstrates it with `serde_json` only.
- `ts-rs` beyond its version number, and `tauri-specta` beyond its README and release-candidate status.
- Clippy's `disallowed-types` configuration for banning `HashMap` in the simulation crate.
- Whether `tauri dev` restarts on a change to a non-Rust file under `src-tauri/`. The docs say it watches the folder for changes and describe `.taurignore`; I did not run it.
- The statements about TOML's verbosity for deep nesting, JSON lacking comments, YAML's typing pitfalls, and RON's speed relative to JSON.
- No code was compiled or run. API names are taken from the sources as read; signatures should be re-checked when the first prototype is written.

## Sources

All read on 2026-11-17. "Tarball" means the published crate archive downloaded from `static.crates.io` and read locally.

- **[S1]** This repository at commit `0d38167`: `src-tauri/Cargo.toml`, `package.json`, `bun.lock`, `src-tauri/src/lib.rs`, `src-tauri/tauri.conf.json`; local `rustc --version`.
- **[S2]** crates.io API, `https://crates.io/api/v1/crates/<name>`, for every crate in the version table (newest stable version, newest version, last update date).
- **[S3]** Tauri 2 docs, State Management. Source: <https://raw.githubusercontent.com/tauri-apps/tauri-docs/v2/src/content/docs/develop/state-management.mdx> (branch `v2`). Rendered at <https://v2.tauri.app/develop/state-management/>.
- **[S4]** Tauri 2 docs, Calling Rust from the Frontend. Source: <https://raw.githubusercontent.com/tauri-apps/tauri-docs/v2/src/content/docs/develop/calling-rust.mdx>. Rendered at <https://v2.tauri.app/develop/calling-rust/>.
- **[S5]** Tauri 2 docs, Calling the Frontend from Rust. Source: <https://raw.githubusercontent.com/tauri-apps/tauri-docs/v2/src/content/docs/develop/calling-frontend.mdx>. Rendered at <https://v2.tauri.app/develop/calling-frontend/>.
- **[S6]** Tauri 2 docs, Develop (source-change watching, `.taurignore`, committing `Cargo.lock`) and Embedding Additional Files. Sources: <https://raw.githubusercontent.com/tauri-apps/tauri-docs/v2/src/content/docs/develop/index.mdx> and <https://raw.githubusercontent.com/tauri-apps/tauri-docs/v2/src/content/docs/develop/resources.mdx>. Rendered at <https://v2.tauri.app/develop/> and <https://v2.tauri.app/develop/resources/>.
- **[S7]** `tauri` 2.12.6 tarball, <https://static.crates.io/crates/tauri/tauri-2.12.6.crate>: `src/ipc/channel.rs`, `src/lib.rs` (`Manager::manage`), `src/path/desktop.rs`, `Cargo.toml`.
- **[S8]** The same `tauri` 2.12.6 tarball: `src/test/mod.rs` (module documentation).
- **[S9]** `tauri-specta` README, <https://raw.githubusercontent.com/specta-rs/tauri-specta/main/README.md> (branch `main`).
- **[S10]** `bevy_ecs` 0.20.1 README and manifest, <https://raw.githubusercontent.com/bevyengine/bevy/v0.20.1/crates/bevy_ecs/README.md> and `.../crates/bevy_ecs/Cargo.toml` (tag `v0.20.1`).
- **[S11]** `bevy_ecs` 0.20.1 source at tag `v0.20.1`: `crates/bevy_ecs/src/system/query.rs`, `src/hierarchy.rs`, `src/entity/mod.rs`, under <https://raw.githubusercontent.com/bevyengine/bevy/v0.20.1/>.
- **[S12]** `hecs` 0.11.2 README, manifest, and changelog, <https://raw.githubusercontent.com/Ralith/hecs/master/README.md>, `.../Cargo.toml`, `.../CHANGELOG.md` (branch `master`, whose manifest reads 0.11.2).
- **[S13]** `hecs` source on `master`: `src/world.rs`, `src/serialize/row.rs`, `src/serialize/column.rs`, under <https://raw.githubusercontent.com/Ralith/hecs/master/>.
- **[S14]** The Rust Rand Book, chapters Reproducibility, Updating to 0.10, Seeding RNGs, Our RNGs. Sources: <https://raw.githubusercontent.com/rust-random/book/master/src/crate-reprod.md>, `.../update-0.10.md`, `.../guide-seeding.md`, `.../guide-rngs.md`. Rendered at <https://rust-random.github.io/book/crate-reprod.html>.
- **[S15]** `slotmap` 1.1.1 crate docs and manifest, <https://raw.githubusercontent.com/orlp/slotmap/master/src/lib.rs> and `.../Cargo.toml` (branch `master`, manifest reads 1.1.1).
- **[S16]** `rand` 0.10.4 on `master` (manifest reads 0.10.4): <https://raw.githubusercontent.com/rust-random/rand/master/Cargo.toml>, `.../src/rngs/mod.rs`, `.../src/rngs/xoshiro256plusplus.rs`, `.../CHANGELOG.md`.
- **[S17]** `rand_pcg` 0.10.2 and `rand_xoshiro` 0.8.1 on `master`: <https://raw.githubusercontent.com/rust-random/rngs/master/rand_pcg/Cargo.toml>, `.../rand_pcg/src/lib.rs`, `.../rand_pcg/src/pcg64.rs`, `.../rand_xoshiro/Cargo.toml`.
- **[S18]** `rand_chacha` 0.10.0 tarball (<https://static.crates.io/crates/rand_chacha/rand_chacha-0.10.0.crate>: `README.md`, `Cargo.toml`, `src/chacha.rs`) and `chacha20` 0.10.0 tarball (<https://static.crates.io/crates/chacha20/chacha20-0.10.0.crate>: `src/rng.rs`, `Cargo.toml`).
- **[S19]** Rust standard library at tag `1.99.1`: `library/std/src/collections/hash/map.rs`, `library/alloc/src/collections/btree/map.rs`, `library/core/src/primitive_docs.rs` (the `f32` section), `library/std/src/num/f64.rs`, `library/std/src/fs.rs`, under <https://raw.githubusercontent.com/rust-lang/rust/1.99.1/>.
- **[S20]** `indexmap` README, <https://raw.githubusercontent.com/indexmap-rs/indexmap/main/README.md> (branch `main`; newest release 2.14.2).
- **[S21]** `ron` 0.12.3 at tag `v0.12.3`: <https://raw.githubusercontent.com/ron-rs/ron/v0.12.3/README.md>, `.../Cargo.toml`, `.../src/error.rs`, `.../CHANGELOG.md`.
- **[S22]** `toml` 1.2.1 on `main` (manifest reads `1.2.1+spec-1.1.0`): <https://raw.githubusercontent.com/toml-rs/toml/main/crates/toml/Cargo.toml> and `.../crates/toml/src/de/error.rs`.
- **[S23]** `serde_json` `src/error.rs`, <https://raw.githubusercontent.com/serde-rs/json/master/src/error.rs> (branch `master`; newest release 1.0.154).
- **[S24]** `serde_json` `src/ser.rs`, `src/de.rs`, and `Cargo.toml` on `master`, under <https://raw.githubusercontent.com/serde-rs/json/master/>.
- **[S25]** serde documentation, Container attributes. Source: <https://raw.githubusercontent.com/serde-rs/serde-rs.github.io/master/_src/container-attrs.md>. Rendered at <https://serde.rs/container-attrs.html>.
- **[S26]** `serde_path_to_error` README, <https://raw.githubusercontent.com/dtolnay/path-to-error/master/README.md> (newest release 0.1.20).
- **[S27]** `serde_yaml` README, <https://raw.githubusercontent.com/dtolnay/serde-yaml/master/README.md>.
- **[S28]** `serde-saphyr` 2.1.1 tarball, <https://static.crates.io/crates/serde-saphyr/serde-saphyr-2.1.1.crate>: `README.md` (opening section only).
- **[S29]** `notify` 8.2.2 tarball (<https://static.crates.io/crates/notify/notify-8.2.2.crate>: `src/lib.rs`, `Cargo.toml`), `notify-debouncer-full` 0.7.0 tarball (<https://static.crates.io/crates/notify-debouncer-full/notify-debouncer-full-0.7.0.crate>: `src/lib.rs`, `Cargo.toml`), and the repository README on `main`, <https://raw.githubusercontent.com/notify-rs/notify/main/README.md> (which describes the 9.0 release candidate).
- **[S30]** `tauri-plugin-fs` 2.7.1 tarball, <https://static.crates.io/crates/tauri-plugin-fs/tauri-plugin-fs-2.7.1.crate>: `Cargo.toml`.
- **[S31]** `bincode` 3.0.0 tarball, <https://static.crates.io/crates/bincode/bincode-3.0.0.crate>: `README.md`, `src/lib.rs`.
- **[S32]** `postcard` 1.1.3 tarball, <https://static.crates.io/crates/postcard/postcard-1.1.3.crate>: `README.md`, `src/error.rs`, `src/de/deserializer.rs`, `Cargo.toml`. `rmp-serde` 1.3.1 tarball, <https://static.crates.io/crates/rmp-serde/rmp-serde-1.3.1.crate>: `README.md`.
