# Plan: Jaric Engine (2D -> 3D -> shaders), CLI + makej + editor

> **LOCKED by the owner on 2026-10-23. The AI must not change this plan; only the owner can (record changes below as "owner change").**

## Context
Jcmp 1.2.0.0 is released. The owner wants a game engine on J2K: easy like C# / MonoGame, able to make Terraria / Stardew-class 2D games and then 3D with good shaders, data-oriented (SoA hot data, AoS records), usable raw and high-level in one program, one library `#import <gamej>`.
**Rule from the owner: once this plan is approved, the AI must not change it.** Changes only by the owner (record them in `Jaric-Engine/docs/engine-plan.md` as "owner change"). Language syntax questions still go to `docs/open-questions.md` (never decided by the AI).

## Facts that shaped the plan
* Speed of J2K vs `clang -O2` (docs/PROJECT_SUMMARY.md): fib 1.0x, sieve 0.9x, matmul 1.0-1.2x, mandel ~1.0x, gcd 1.0x, nbody 1.5x, sha256 1.1x, hashmap 1.2x, vec3 1.2-1.8x, simple loops that clang turns into SIMD up to 4x. So: **not slower than C++ in general; behind where SIMD matters.** Engine answer: hot data in SoA, loops with `<simd>`, heavy work on the GPU (compute), measured in benchmarks, not guessed.
* Existing pieces (Jaric-Engine repo): `libs/gamej.j` (CPU pixels, terminal + surface), `gpu.j`, `opengl3.j` (GLES 3.1 + compute, native), `vulkan.j` (info), `native.j`. App protocol `7777;gfx/gpu/audio`. Inheritance (struct : Base, virtual/override) exists since 1.2.0.0 — for scene nodes / tools only, not for hot loops.
* Owner's answers: tools = **everything: CLI (basic commands complete), makej, and an editor**; order = **2D, 3D, shaders, step by step**.

## Decisions (locked on approval)
1. One import: `#import <gamej>`; parts are modules inside it (`Tile::`, `Ecs::`, `Phys::`, `Path::`, `Fx::`, `Ui::`, `Gfx3::`). `gpu opengl3 vulkan native` stay separate = raw layer; every engine object exposes `.handle`.
2. API in MonoGame style (`Game` load/update/draw, `SpriteBatch`, `Content.load`, input states). Targets: hello-sprite <= 25 lines, Pong <= 80.
3. Data: SoA pools for entities/components (ECS-lite: entity = index, dense array + sparse index, delete = swap with last); AoS for vertices/records; per-frame Arena.
4. Tools: `jaric-engine` (alias `jengine`) CLI: `doctor info install new run pack` (+ later `scene`, `assets`, `bench`); projects built by `makej` (`makej init -game`); editor last (on the Jaric app, scene = text `.jscn`, editable by coderin from day 1).
5. Path: GLES 3.1 through `<opengl3>` (`-dyn`, works now); Vulkan later; the app surface path stays for the apk.

## Stages (each ends with tests + docs + commit + push; small = 1 test round)
**E0 Base** — merge libs into one `gamej.j` (gen script joins parts), keep t510/t511 green; `jaric-engine doctor` + `info` (read-only); benchmark SoA vs AoS (100k sprites) written to `docs/dod-and-installer.md`.
**E1 2D core** — `Game`, `SpriteBatch` on GLES (batching, textures, tint, rotation, atlases), PNG via `<deflate>` + decoder, bitmap fonts, `Content`, input states (touch/keys), camera 2D, sound via app OSC. Examples: hello sprite, Pong.
**E2 2D worlds** — `Ecs` pools + systems, `Tile` (chunked SoA layers, dirty flags, save/load), `Phys` (AABB vs tiles, platforms), `Path` (A*), `Clock` (day/night), `Ui`. Examples: mini-Terraria (dig/place/gravity, 2000x600), mini-Stardew (farm, crops by day, NPC).
**E3 Tools** — `jaric-engine new/run/pack` over makej; `.jscn` scene text + loader; `scene` and `assets` commands; `install` (jcmp, makej, libs; apk/android.jar only for packing; asks before downloading the app).
**E4 3D** — `Camera3D`, `Mesh/Model` (obj, then gltf), `Material`, lights, depth, spinning lit cube, shadow maps; MSAA.
**E5 Shaders** — GLSL files in `shaders/` with hot reload, default set (`lit pbr toon water bloom outline crt`), post-process chain, PBR + normal maps + skybox + HDR/bloom/tonemap; compute-shader particles (`Fx`).
**E6 Editor** — scene tree view, drag/drop, tile painter, live run, on the Jaric app (protocol extension, owner reviews the screen design first).

## Files / reuse
Repo `~/Jaric-Engine`: `libs/gamej.j, gpu.j, opengl3.j, vulkan.j, native.j` (reuse), new `tools/jaric-engine/*.jk`, `docs/engine-plan.md` (this plan, copied on approval), `docs/dod-and-installer.md` (exists), `examples/`. Library source copy in `~/Jcmp/std` only if the std set is extended (`gen_std.sh`), Jcmp-libs index updated per release. Tools repo `~/jget` (makej templates for `init -game`). Public docs never link private files.

## Verification
Per stage: example runs on the phone (`-dyn` GLES path and terminal path), a test in `Jcmp/test/` for library behaviour (registered in `run_all_tests.sh`), `JCMP=... ./run_all_tests.sh` all PASS, benchmark numbers recorded (SoA vs AoS, sprites per second at 60 fps, tiles per second), tutorial lesson per stage, push `private dev` / Jaric-Engine origin after every step.

## Not in this plan
Own shader language, own kernel/GPU driver, Vulkan renderer parity (after E5), any change to J2K syntax (open-questions only).

## Owner change 2026-10-23
`jengine setup NAME` makes a whole project at once with several modes (`-2d -3d -term -pro`): folders `src/ assets/ lib/ shaders/ scenes/`, starter code, and a `makej.toml` so `makej` / `makej -j` / `makej run` work at once. It belongs to E3 (`new` is its alias); the starter code and the modes are made when E3 starts.
