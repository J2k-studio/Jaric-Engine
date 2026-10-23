# Jaric Engine — where it is and where it goes  (2026-10-23; proposal, the owner decides)

## The layers
| Layer | What | Library | State |
|---|---|---|---|
| 0 Chip | talk to the graphics chip: triangles, textures, shaders, compute | `<gpu>` (OpenGL ES in the app) → Vulkan / OpenCL through `<native>` | gpu 0.1.0 done (GLES); `native` spike works (Vulkan 1.1 answers from J2K) |
| 1 Framework | the loop, input, time, 2D drawing, sound, assets | `<gamej>` | 0.2.0 done (CPU pixels; terminal + surface) |
| 2 Scene | nodes in a tree, scenes in files, signals, resources, tilemaps, sprites with animation, collision shapes, a camera | `<jscene>` | not started |
| 3 Tools | an editor (scene, tiles, animation), a profiler, an asset importer | `jedit` | not started |
| 4 Export | one project to Android / Jaric OS / Linux | `makej dist` | done for the three |

## Which model to follow
| | .NET family (XNA / MonoGame / Unity) | Unreal | Godot |
|---|---|---|---|
| Idea | MonoGame: a *framework* (you write the loop); Unity: components on objects | a huge C++ engine, blueprints, editor first | a small engine; everything is a **node**; scenes are trees saved in text; a language made for it (GDScript) |
| Size / cost | MonoGame small, Unity big | enormous (GBs), needs a strong PC | small (about 40 MB), runs on a phone |
| For us | **layer 1** is MonoGame / raylib: already so | wrong size for a phone and for a language we write ourselves | **layer 2** fits: nodes + scenes + signals; J2K can be the script language (no new language) |
| Verdict | follow for the framework | do not follow (take only the idea of a renderer with materials later) | **follow for the scene system and the editor** |
**Recommendation:** layers 0 and 1 are like MonoGame (low level, you hold the loop; the GPU API under it like a small `GraphicsDevice`); layer 2 like Godot (a tree of nodes, scenes as text files `.jscn`, signals, groups, a tile map, an animation player, the J2K file of a node is its script); never Unreal's size. Ownership of J2K (no GC) makes the node tree simple: a parent owns its children.

## The GPU plan, in order
1. **Now (works):** `<gpu>` through the app: OpenGL ES 2.0, one batch per frame, textures, tint, alpha.
2. **Next:** GLES 3.1 *compute shaders* (`GLES31.glDispatchCompute`) through the app: the program sends GLSL and buffers; for physics, particles, matrices, small networks. No native code.
3. **Then:** `<native>` (dlopen / dlsym from memory, calls by function pointer) and `<vulkan>` on it: instance, device, swapchain on the Surface, a triangle, then pipelines, then the same API as `<gpu>` under it; OpenCL the same way for compute. Only in the process of the app (a `-l` library).
4. **Later:** shaders written in J2K (a small shader language that compiles to SPIR-V / GLSL) — an owner's decision, a syntax question.

## The engine now, honestly
Framework (layer 1): ~70% (loop, input, drawing, text, art, collision, sound, touch; missing: images from files on the CPU path, sprite sheets, tile maps, fonts, many voices). Chip (layer 0): ~25%. Scene (layer 2): 0%. Tools: 0%. Export: 90% (the apk runner does not know `<gpu>` yet).

## Next steps
1. `<gamej>` draws through `<gpu>` in the app; the runner of the apk learns `<gpu>`; images (png) by `<deflate>` + a PNG decoder; sprite sheets and tile maps.
2. `<jscene>` v0: `Node`, `Node2D`, `Sprite`, `Camera`, `Timer`, the loop calls `_process(dt)` of each node; a scene file `.jscn` (text).
3. Compute shaders; then `<native>` and the Vulkan triangle.

## Update (owner, 2026-10-23): not "easy": a modern MonoGame-style engine; and native GPU without the app
**API in the style of MonoGame** (the owner's choice; a draft to review, nothing locked). The parts: `Game` (the loop with `Initialize`, `LoadContent`, `Update(GameTime)`, `Draw(GameTime)`, `UnloadContent`), `GraphicsDevice` (clear, viewport, render targets, shaders), `SpriteBatch` (`Begin(sort, blend, sampler, transform)`, `Draw(texture, position/rectangle, source, color, rotation, origin, scale, effects, depth)`, `DrawString`, `End`), `Texture2D`, `SpriteFont`, `ContentManager` (`Load<T>(path)`), `Effect` (shaders), math (`Vector2/3/4`, `Matrix`, `Rectangle`, `Color`, `MathHelper`, `Quaternion`), input (`Keyboard`, `Mouse`, `TouchPanel`, `GamePad` as *states* you read each frame), `SoundEffect`/`Song`, `GameWindow`. J2K has no inheritance, so the draft uses a struct that holds the callbacks (function pointers) or a struct that you extend by composition:
```
struct MyGame { Texture2D player; Vector2 pos; }
void load(MyGame^ g, Content^ c) { g^.player = c^.load_texture("player.png"); }
void update(MyGame^ g, GameTime t) { if Keyboard::state().is_down(Key::Left) { g^.pos.x -= 200.0 * t.dt; } }
void draw(MyGame^ g, SpriteBatch^ sb, GameTime t) { sb^.begin(); sb^.draw(g^.player, g^.pos, Color::white()); sb^.end(); }
int main() { Game game = Game::create("My Game", 1280, 720); game.run(load, update, draw, MyGame()); }
```
(the exact spelling is the owner's to choose). Under it: `SpriteBatch` collects quads into a vertex buffer and sends **one** draw call per texture (the command buffer of `<gpu>` today; Vulkan later), so it is also the fast one.
**Native GPU, not through the app** — three levels:
1. *Through the app* (works): GLES by the app, commands in a file.
2. *A standalone program that calls the driver libraries itself* (`libvulkan.so`, `libGLESv3.so`, `libOpenCL.so` of the phone): needs the program to be a **dynamic executable** (an ELF with `PT_INTERP /system/bin/linker64`, `DT_NEEDED libc.so libdl.so`, and relocations for `dlopen` / `dlsym`). The compiler already writes a dynamic ELF for `-l` (`.dynsym`, `.hash`, `.dynamic`); the missing part is the interpreter, `DT_NEEDED`, and `GLOB_DAT` relocations for two imports. A new flag `jcmp -dyn` (a few hundred lines). Then a normal program (no app) runs Vulkan compute and renders off screen; to put the picture on the screen it sends the pixels through `gfx` (a copy) or, later, shares a buffer. No display exists in Termux by itself (termux-x11 is another way).
3. *Our own driver*: `/dev/mali0` opens from Termux on this phone (mode 666). A program could talk to the kernel driver (kbase ioctls) and send its own command streams, as Mesa Panfrost does: the Bifrost instruction set and a shader compiler would be written by us. This is the true "native GPU of Jaric" and a very long project (years of work in Mesa); it belongs to Jaric OS (K track) when the OS owns the driver. Not now.
**Recommendation:** level 2 next (`jcmp -dyn`, then `<native>` + `<vulkan>`), level 1 stays as the app path, level 3 is a long-term goal of the OS.
