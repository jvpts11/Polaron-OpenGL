# Polaron-OpenGL

A modern **OpenGL 4.6** library for **Polaron**, reached entirely through the language's FFI — no C, no
third-party glue, no bindings generator. It is a real Polaron `[library]` project: `polaron build`
compiles it to a distributable bundle (`Polaron-OpenGL.polb` + `.polh`) that a program installs with
`polaron plug` and `import`s.

```
polaron plug https://github.com/jvpts11/Polaron-OpenGL@v1.0.0
```

## Why a library

`opengl32` exports only OpenGL 1.1 directly. Every modern entry point — `glCreateShader`,
`glGenVertexArrays`, `glUniformMatrix4fv` — has to be loaded at runtime with `wglGetProcAddress` and
called through a function pointer. Polaron has a language feature for exactly this: **`funcptr<Ret,
Args...>`**, a bare C function pointer with no closure environment. This library wraps the whole dance so
consumers never touch it.

The window is real too: `RegisterClassEx` with the OS's own `WNDPROC`, a `WS_OVERLAPPEDWINDOW` that
drags, resizes and closes like any application window, and a 4.6 core context built through
`wglCreateContextAttribsARB`.

## What is in it

Six namespaces, one per role, each the equivalent of a library the C world reaches for:

| Namespace | Role | Analogue |
|-----------|------|----------|
| `OpenGL.Core` | Every modern entry point as a `funcptr<>` field of `Gl`, loaded by `Gl.load()`; the GL 1.1 core and WGL on `Wgl`; pixel-format selection on `Gdi`; the enumerants on `GlEnum`. | GLEW + GL |
| `OpenGL.Windowing` | `GlWindow` — a native window with a 4.6 core context, an event pump, a keyboard event queue, and polled mouse/scroll input. | GLFW |
| `OpenGL.Geometry` | `GlMat4` / `GlVec3` column-major math (perspective, ortho, lookAt, translate, rotate, scale) in buffers ready for `glUniformMatrix4fv`. | GLM |
| `OpenGL.Textures` | `Texture.load2D` / `load2DRGBA` — decode an image with the decoder Windows already ships (GDI+) and upload it. | stb_image |
| `OpenGL.Shaders` | `Shader.fromFiles` / `fromSource` — compile and link GLSL programs, with the driver's info log on failure. | — |
| `OpenGL.Text` | `GdiFont.rasterizeAscii` — rasterize a monospace face into a glyph atlas with grayscale anti-aliasing. | FreeType, for the ASCII case |

## Using it

`polaron plug` records the dependency for you; then, per file:

```polaron
import OpenGL.Core.Gl;
import OpenGL.Core.Wgl;
import OpenGL.Core.GlEnum;
import OpenGL.Windowing.GlWindow;
import OpenGL.Geometry.GlMat4;
import OpenGL.Shaders.Shader;
```

```polaron
mutable GlWindow* win = new GlWindow() on heap;
if (!win.open("demo", 1280, 720, 4, 6)) {
    System.IO.Console.println("no OpenGL 4.6 on this machine");
    delete win;
    return;
}
win.setVSync(1);
while (win.isOpen()) {
    win.pump();
    Wgl.glClear(GlEnum.COLOR_BUFFER_BIT | GlEnum.DEPTH_BUFFER_BIT);
    // ... draw ...
    win.swap();
}
delete win;
```

The consumer's manifest needs nothing about opengl32, gdi32 or the rest: each class here names the
foreign library it comes from with `class Wgl library OpenGL`, and this project's manifest maps that
logical name to the file each platform spells it in.

## Requirements

An OpenGL 4.6 driver (any modern GPU) and Windows x64.

## Converted from ldp3-opengl

This is the LDP3-era `ldp3-opengl` brought forward to Polaron, and the move was not a rename. What
changed, and why:

- **One type per file.** Six files holding twenty-one types became twenty-one files. A reader looking for
  `GlWindow` opens `GlWindow.pol`.
- **Namespaces that say what they are.** `gl`, `glfw`, `glm`, `glsl`, `gltex`, `gltext` were the names of
  the C libraries each one replaces. `Core`, `Windowing`, `Geometry`, `Shaders`, `Textures` and `Text`
  are what they do.
- **`extern cdecl`, not `extern stdcall`.** Polaron's extern axis is the foreign *language*, not the
  calling convention — the convention is an ABI detail, and on x86-64 the three old convention words all
  collapsed onto the one platform ABI.
- **`class X library Y`.** The library says which foreign library its externs come from. Consumers used
  to have to copy `native_libs = "opengl32, gdi32, user32, kernel32, gdiplus"` into their own manifest,
  which is this library's private business written down in somebody else's file.
- **`address` at the FFI boundary.** A `Rect*` handed to `GetClientRect` is a pointer nothing in the
  language can follow: past the extern, no analysis can say whether the OS keeps it. `address` is how a
  program says the lifetime is outside the language, and the region binder refuses the alternative.
- **`open` answers `boolean`.** It returned an `int` that was 1 on success, and a consumer once read it
  as a status code — the window appeared, the check declared failure, the program fell back to software
  rendering and exited: "the window opened and closed". A boolean cannot be read backwards, and every old
  call site comparing against 0 now fails to compile.
- **`CursorKind` and `EventKind`.** `setCursor(3)` used to be a legal thing to write. Two enums replaced
  an `int` carrying 0–4 and an `int` carrying 0, 1 and -1.
- **`Gl` travels as `Gl*`.** It holds fifty function pointers, and Polaron assignment is a deep copy, so
  every `Shader.compile(gl, ...)` used to copy all fifty.
- **Leaks closed.** `Shader.readFile` dropped its `StringBuilder` and its line list on every call;
  `FontAtlas` never freed its pixels; `GlWindow` never freed its four native buffers. The window now owns
  what it allocates and releases it in a destructor.
- **The two image decode paths became one.** They were forty identical lines apart from two constants;
  the three values they had to return are now a `DecodedImage`.
