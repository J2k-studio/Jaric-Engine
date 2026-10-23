# Jaric Engine

A game engine and graphics libraries for the [J2K language](https://github.com/J2k-studio/Jcmp), written in J2K. It runs on a phone (Termux and the Jaric app) and on any Linux ARM64 machine.

| library | what it is | needs |
|---|---|---|
| **`<gamej>`** | the framework: the loop, input, time, 2D drawing, text, pictures made of letters, collision, sound, touch | draws in a terminal (half blocks, 24-bit colour) or on a surface in the Jaric app |
| **`<gpu>`** | the graphics chip through the Jaric app: triangles, rectangles, lines, textures, tint, alpha, one batch of commands per picture (OpenGL ES in the app) | the Jaric app |
| **`<native>`** | call the libraries of the system from J2K (`dlopen`, `dlsym`, C calls, float arguments, memory) | a program built with `jcmp -dyn` |
| **`<opengl3>`** | OpenGL ES 3.1 / 2.0 on the real driver: shaders (GLSL), triangles, **compute shaders**, read the pixels back | `jcmp -dyn` and `<native>` |
| **`<vulkan>`** | Vulkan on the real driver (first step: instance, the chip, memory, queues); the rest is built with `<native>` | `jcmp -dyn` and `<native>` |

## Install
```
jget -li install gamej gpu native opengl3 vulkan      # the libraries (jget: https://github.com/J2k-studio/Jcmp-Tools)
makej init -game                                      # a game project with Jaric Engine and five lessons (makej play1 .. play5)
makej apk                                             # the game as an Android app;  makej pack = .jec;  makej dist = every system
```
`jcmp` 1.1.1 or newer (`jget tool jcmp`).

## A game (about ten lines)
```jk
#import <stdlib>
#import <gamej>
int main() {
    Game g = Game::open("Lesson");
    double x = 20.0;
    while g.running() {                       // false after Esc
        g.begin();                            // keys, touches, g.dt (seconds)
        if g.held(Key::left()) { x -= 40.0 * g.dt; }
        if g.held(Key::right()) { x += 40.0 * g.dt; }
        g.clear(Game::rgb(10, 20, 40));
        g.rect((int)x, 30, 6, 6, Game::rgb(255, 200, 60));
        g.end();
    }
    g.close();
    return 0;
}
```

## The chip itself (OpenGL ES 3.1, compute shaders, Vulkan)
```
jcmp -dyn examples/opengl3_triangle_and_compute.jk -o gl && ./gl      # triangle + a compute shader on the Mali-G76
jcmp -dyn examples/vulkan_info.jk -o vk && ./vk                       # Mali-G76 Vulkan 1.1.97 ...
```
How it works: a program built with `-dyn` is started by the loader of Android and gets `libdl.so`; `<native>` finds `dlopen`/`dlsym` in memory and calls any function of `libEGL.so`, `libGLESv3.so`, `libvulkan.so` through function pointers. No app and no C code is needed.

## The plan
Layers (like MonoGame below, like Godot above): 0 the chip (`<gpu>`, `<opengl3>`, `<vulkan>`), 1 the framework (`<gamej>`), 2 scenes and nodes (planned), 3 tools (editor, planned), 4 export (`makej dist`: Android, Jaric OS `.jec`, Linux). See [docs/roadmap.md](docs/roadmap.md). Tutorial in the J2K repository: lessons 61 to 66.

License: MPL-2.0.
