# Jaric Engine: data-oriented core and the `jaric-engine` program  (2026-10-23; designed by AI at user request, adjustable)

## 1. Data-oriented design (DOD): SoA for hot data, AoS for small records
Rule: **data that a loop touches for many items lives in separate arrays per field (SoA); a record that is read as a whole lives in a struct array (AoS).**
| Use | Layout | Why |
|---|---|---|
| positions, velocities, colours, sprite frames of 10 000 things | **SoA**: `x[] y[] vx[] vy[]` | a loop over `x[] += vx[] * dt` reads two arrays in a row, fits the cache, and the compiler's NEON loops (`<simd>`) can take 4 lanes |
| a vertex (x, y, u, v, colour) sent to the GPU | **AoS**: one struct per vertex | the GPU reads a whole vertex at once; one `mmap`ed buffer, one upload |
| a level tile, an item description | AoS | read as a whole, rarely in bulk |
| the entities | **ECS-lite**: an entity is an index; a *component pool* is a SoA table (`Pool<Position>`, `Pool<Sprite>`); a *system* is a plain function over one or two pools | no pointer chasing, no virtual call per entity, easy to run on many cores |
J2K support that exists today: struct arrays, `Floats`/`Ints` buffers, `<simd>` loops, `-j` is for the compiler only. Inheritance (1.2) is for tools and UI nodes (scene layer), **not** for the hot path.

Library plan (`<jecs>`, layer 1.5): `World`, `Pool<T>` (dense array + `sparse[]` index + `entity[]` back-index, delete = swap with last), `Query2<A,B>` iterate two pools, `Systems` run in order each frame. Frame memory: an `Arena` that is reset every frame (see Jcmp plan A3).

## 2. The program `jaric-engine`
Name: **`jaric-engine`** (command), short alias `jengine`. (`enginejar` is not used: `.jar` means Java; the owner can still rename — open question.)
| Command | Does |
|---|---|
| `jaric-engine doctor` | checks what the phone has and what is missing; prints one line per item with OK / MISSING and how to fix |
| `jaric-engine install` | installs the minimum: `jcmp`, `makej`, `jget`, the libraries `gamej gpu opengl3 vulkan native vec simd deflate`, the Jaric app (apk) if the user wants the surface path |
| `jaric-engine new NAME [-2d|-3d|-term]` | makes the project (calls `makej init -game`), with `src/ assets/ shaders/ scenes/ makej.toml` |
| `jaric-engine run` | builds (`makej -j`) and starts; on the phone uses the app surface or the native `-dyn` path |
| `jaric-engine pack` | `.jec` / apk through `makej dist` |
| `jaric-engine info` | GPU name, GLES / Vulkan version, compute support, cores, screen size |

## 3. What "everything needed" means (the minimum, per layer)
| Need | Provided by | Where it comes from |
|---|---|---|
| CPU compile | `jcmp` | jget / install script |
| GPU, OpenGL ES 2/3.1 + compute | `<opengl3>` through `<native>` (`-dyn`) | the phone's `libEGL`/`libGLESv3` (system) |
| GPU, Vulkan 1.1 | `<vulkan>` | the phone's `libvulkan` (system) |
| GPU through the app (no native code) | `<gpu>` + app GpuView | the Jaric app apk |
| 2D/3D graphics API (gfx) | `<gamej>` (surface, terminal) | Jaric-Engine libs |
| visual effects (vfx): particles, trails, post-process | `<jvfx>` (planned): particle pools in SoA + compute shader | Jaric-Engine libs |
| sound | app audio OSC (`7777;audio`) | the app |
| video | `jvp4` | Jcmp-Tools |
| input (touch, keys) | `<gamej>` | the app / terminal |
| Android pieces (`androidjar`) | only `android.jar` of the SDK **for packing an apk**; `jaric-engine install -apk` fetches it once; the game itself needs none | Android SDK / Termux package |
"gpx" is read as the GPU command-file format of `<gpu>` (the 7777;gpu protocol), "glse" as OpenGL ES.

## 4. Order of work
1. `jaric-engine doctor` + `info` (read-only, safe) — in J2K, in `Jaric-Engine/tools/`.
2. `<jecs>` Pool/Query + a benchmark: SoA vs AoS loop over 100 000 sprites (numbers go in this file).
3. `new` / `run` / `pack` over `makej`.
4. `install` (needs network + the owner's choice about the apk).
5. `<jvfx>` particles on compute shaders.

## Open questions for the owner
* Name: `jaric-engine` or something else? * Should `install` download the app apk, or only tell where it is?
* Is `<jecs>` the name for the ECS library?

## 5. Targets set by the owner (2026-10-23)
**Ease:** writing a game must be as easy as in C# with MonoGame: `Game` with `load / update / draw`, `SpriteBatch`, `Content.load`, input as state objects, no manual memory for the common case (ownership frees it). Test: the "hello sprite" game is at most 25 lines, a Pong at most 80.
**Breadth:** the engine must be able to make games like **MonoGame samples, Terraria and Stardew Valley** (2D, tile worlds, many entities). What those games need, and where it comes from:
| Need | Terraria-like | Stardew-like | Piece |
|---|---|---|---|
| big tile world, chunks, lazy load/save | yes (8400x2400) | yes (maps) | `<jtile>`: chunked SoA tile layers (`u16` id, light, liquid), dirty flags, save to file |
| tile collision, slopes, platformer physics | yes | grid only | `<jphys2d>`: AABB vs tiles, one-way platforms, rigid bodies later |
| lighting (flood light), liquids, falling sand | yes | no | cellular passes in SoA, optional compute shader |
| sprite animation, atlases, 9-slice UI, bitmap fonts | yes | yes | `<gamej>` + `<jui>` |
| top-down tiles, NPC paths, day/night clock, seasons, inventory | little | yes | `<jpath>` (A*), `<jclock>`, ECS components (`<jecs>`) |
| items/recipes/data in files | yes | yes | JSON/TOML (stdlib) read at load |
| camera, parallax, particles, sound, save games, touch controls | yes | yes | `<gamej>`, `<jvfx>`, app audio, `Fs`, touch |
| many entities (thousands of NPCs, projectiles, drops) | yes | some | SoA pools, systems over pools, many cores |
Order after the core: `<jtile>` + tile collision -> a "mini Terraria" example (dig, place, gravity, 2000x600 world) -> `<jclock>`/`<jpath>` -> a "mini Stardew" example (farm tiles, crops that grow by day, NPC walking). Each example doubles as a test and a tutorial.

## 6. One library, 3D and shaders, raw access (owner, 2026-10-23; supersedes the many names above)
* **One import:** `#import <gamej>` gives everything of the engine (the existing way to import; a `using`-style keyword would be a syntax decision, so it goes to open-questions). The planned `jecs jtile jphys2d jpath jclock jvfx jui` are **parts (modules) inside gamej**, not separate libraries: `Tile::`, `Ecs::`, `Phys::`, `Path::`, `Fx::`, `Ui::`. Files may stay split in the repo, but `gen` joins them into one `gamej.j`; the user sees one name. `gpu opengl3 vulkan native` stay separate because they are the raw layer.
* **Mixed with raw:** the same program may use the simple API (`SpriteBatch`, `Model::draw`) and drop to the raw layer at any line (`Gl::`, `Vk::`, compute shaders, `Cpu::inst`): the engine never hides the GPU; every engine object exposes its raw handle (`tex.handle`, `buf.handle`).
* **3D:** `Camera3D`, `Mesh`/`Model` (obj/gltf import), `Material`, lights (directional, point, spot), depth buffer, shadow maps, then PBR (metal/rough), normal maps, skybox, HDR + bloom + tonemapping, MSAA. Runs on OpenGL ES 3.1 first (works today through `<opengl3>`), Vulkan later.
* **Shaders:** a `Shader` is a pair of GLSL files in `shaders/` loaded with `Content.load` and **hot reloaded** while the game runs; the engine ships beautiful defaults (`lit`, `pbr`, `toon`, `water`, `bloom`, `outline`, `crt`); post-process chain as a list. Own J2K-to-shader language stays a later, owner-decided item.
* **Order:** 1) merge into one `gamej` and keep tests green; 2) `Camera3D + Mesh + lit shader` (a spinning lit cube, in `-dyn` native); 3) obj/gltf + textures + shadow map; 4) PBR + bloom; 5) the 2D world parts (tile, ecs) for the Terraria/Stardew examples.
