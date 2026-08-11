# Pathfinder

> Geometry Dash physics simulator and automatic macro generator (Geode mod)

Auto-generates completable macros for Geometry Dash levels using a deterministic
frame-based physics simulation. No game window or bot is required at solve time:
the entire level is simulated offline and the resulting input sequence is exported
as a `.gdr2` replay for playback with an external bot (e.g. Eclipse).

Simulation fidelity: **full support up to Geometry Dash 1.7**, partial up to 1.9.

---

## Table of contents

- [Architecture](#architecture)
- [How it works](#how-it-works)
  - [Physics engine](#physics-engine)
  - [Search algorithm](#search-algorithm)
  - [Replay export](#replay-export)
- [Building](#building)
- [Continuous integration](#continuous-integration)
- [Usage](#usage)
- [Supported / unsupported features](#supported--unsupported-features)
- [Versioning & changelog](#versioning--changelog)
- [License](#license)

---

## Architecture

The repository is split into three components:

| Component        | Path                | Role                                                                 |
| ---------------- | ------------------- | -------------------------------------------------------------------- |
| Geode mod        | `src/`              | In-game UI, level extraction, background thread management, export   |
| Physics engine   | `gd-sim/`           | Deterministic Geometry Dash physics simulation, optimised for speed  |
| Pathfinding core | `src/pathfinder.cpp`| Stochastic search over input space driven by the simulation          |

The mod hooks into `EditLevelLayer` and `LevelInfoLayer`, adds a **Pathfinder**
button, and runs the solver in a detached `std::async` task so the game stays
responsive while a progress percentage is shown on screen.

### Dependencies

- [Geode SDK](https://geode-sdk.dev) (nightly) + bindings from `main`
- [gd-sim](https://github.com/camila314/gd-sim) (vendored in `gd-sim/`)
- [UIBuilder](https://github.com/camila314/UIBuilder) (CPM)
- [GDReplayFormat](https://github.com/maxnut/GDReplayFormat) (CPM, `gdr2`)

---

## How it works

### Physics engine

`gd-sim` is a standalone C++20 re-implementation of Geometry Dash's physics,
decoupled from the game. Levels are parsed from their in-memory string format
into an object list (`Level.hpp`). Simulation runs at 240 Hz:

- Frame stepping via `Level::runFrame(bool press)`.
- State snapshots and `Level::rollback(frame)` for branch-and-rewind exploration.
- Deterministic tick for a given input sequence (no game state is required).

### Search algorithm

`pathfind()` (`src/pathfinder.cpp`) performs a stochastic forward search rather
than an exhaustive BFS/A*:

1. At the current frame, generate up to **300 candidate input sets**, each
   containing **30 random press/release frames** within the next 30 frames.
2. For each candidate, `tryInputs` rolls the simulation forward; candidates are
   ranked by furthest progress (`currentFrame`). Positions offscreen (`y > 1300`
   or `y < 0`) are discarded.
3. The best candidate is committed: its press toggles are applied and the
   simulation advances.
4. **Stall recovery**: if no candidate improves progress, the sim rolls back to a
   recent best state (`trueBest`) and retries with fresh randomness, widening the
   retry window (`fail`/`numAway`) until a new frontier is found.
5. The best-known state is snapshotted (`lvlBest`) and becomes the source for the
   exported macro, so the output is never worse than what the solver reached.

The search terminates when `pos.x >= length` or the user stops it. Progress is
reported via a callback as `pos.x / length` (capped at 100%).

### Replay export

The final input stream is derived from consecutive player-state transitions in
`lvlBest.gameStates` and encoded with **GDReplayFormat** into a `.gdr2` binary,
saved through a native file picker (`Eclipse`-compatible location when
`eclipse.eclipse-menu` is installed).

---

## Building

Requires the [Geode CLI](https://github.com/geode-sdk/cli), a C++20 compiler and
CMake >= 3.21.

```bash
geode sdk install nightly     # this mod targets the nightly SDK + main bindings
geode build                   # debug build
geode build -c Release        # optimised build (-O3, hidden visibility)
```

Cross-target builds (from the CI matrix):

```bash
geode build -p win           # Windows
geode build -p mac           # macOS (universal)
geode build -p android32     # Android 32-bit
geode build -p android64     # Android 64-bit
```

The mod requires `geode.node-ids >= v1.18.0` (see `mod.json`).

---

## Continuous integration

`.github/workflows/multi-platform.yml` builds Windows, macOS, Android32 and
Android64 on every push, then packages the resulting `.geode` files into a single
`Build Output` artifact. `multi-platform-debug.yml` produces debug builds.

---

## Usage

1. Open a level (saved or online).
2. Press the blue **Pathfinder** button.
3. Wait for the search; a percentage is shown. *Stop* halts and exports the best
   result found so far.
4. Export the macro as a `.gdr2` file into your bot's replays folder.
5. Import and play back in-game.

---

## Supported / unsupported features

| Supported                              | Unsupported                                  |
| -------------------------------------- | -------------------------------------------- |
| Modes: cube, ship, wave                | Duals, robot, spider, swing                  |
| Slopes (with partial upside-down)      | Inverted slopes, partial rotated objects     |
| Start positions                        | Rotated objects for cube/UFO/ball            |
| Speed portals (incl. 4x)               | Dash orbs, teleport portals                  |
| Orbs/pads: red, black, green           | Any non-visual triggers, modifier blocks     |
| Physics fidelity up to **1.7** (1.9 partial) | All 2.2 objects and triggers           |

> `debug.cpp` (compiled only in debug builds) provides additional solver
> instrumentation.

---

## Versioning & changelog

Version is managed in `mod.json` (`v1.0.0-beta.<n>`). See `changelog.md` for the
release notes of each beta.

## License

Not currently licensed for redistribution. See `about.md` and `support.md`.
