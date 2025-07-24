+++
date = '2025-07-20T09:00:00-05:00'
draft = false
title = 'Lyra Engine Overview'

categories = ["Blog"]
tags = ["Design"]

mermaid = true
comments = true
showLicense = true
showRelated = true
+++

## Introduction

**Lyra-Engine** is my attempt to create a rendering engine.

I have always been fond of creating a complete game / rendering engine from scratch.
In the past, I have made multiple attempts but none of them got very far due to the
lack of experience. **Lyra-Engine** is currently my most promising project.
**Lyra-Engine** uses a modular design, similar to **TheMachinery**, except that
**Lyra-Engine** chooses to use C++, and does not strictly follow C ABI (as it only
introduces more trouble for me at the current stage).

In this post, I will go over the design philosophy and some choices made, starting
with rendering.

## Design

For each module, **Lyra-Engine** defines a struct of APIs, for example:

```cpp
struct WindowAPI
{
    // api name
    CString (*get_api_name)();

    bool (*create_window)(const WindowDescriptor& desc, WindowHandle& window);
    void (*delete_window)(WindowHandle window);

    bool (*bind_window_callback)(WindowHandle window, WindowCallback&& callback);

    bool (*get_window_size)(WindowHandle window, uint& width, uint& height);
    bool (*get_input_state)(WindowHandle window, WindowInputState& state);

    void (*run_in_loop)();
};
```

These APIs will have implementations in individual shared library (DLL), and loaded at runtime
via `dlopen` (*nix) or `LoadLibrary` (Windows). In this way, components are loosely connected
and can be easily swapped out. **Lyra-Engine** also provides a `Plugin` class template as a
helper to load DLLs:

```cpp
using WindowPlugin = Plugin<WindowAPI>;
```

Just having these raw API structs is not going to be user-friendly. **Lyra-Engine** will create
C++ wrappers over these raw APIs so that users gets the benefit of C++ programming.

## RHI (Render Hardware Interface)

All my previous attempts to create a rendering engine has been focused on Vulkan-based engine.
Therefore my abstraction layer is more or less Vulkan-like. To make sure the RHI abstraction
works for most of the graphics APIs, I chose to use [WebGPU](https://www.w3.org/TR/webgpu/)'s
design as a baseline, and add additional features if they are needed. WebGPU API is a subset
of most of the modern rendering APIs. By adopting an existing design, I can avoid designing
an abstraction layer that is too similar to Vulkan.

After spending some time researching on WebGPU, I found WebGPU to be an implicit API that
users don't need to manage resources manually. On the one hand, implementing a resource
tracking API is going to be more complicated. On the other hand, a lot of the overhead
reduction in modern rendering is by removing the resource tracking and leave it to the user
application. Therefore I introduced the barrier layout / access / sync back into the design.
The synchronization complexity could be alleviated by render graph in the next section.

The current plan is to support both Vulkan and D3D12. Metal support is postponed as other parts
of the engine mature.

## RPI (Render Pass Interface)

[Frame Graph](https://www.gdcvault.com/play/1024612/FrameGraph-Extensible-Rendering-Architecture-in)
has been used by Frostbite for many years. It has proven to be useful in managing resources
dependencies. It also allows more modular design when renderer adopts hundred of passes.

## Shading Language

The choice of what shading language to use is an important one. I have been writing GLSL for a long
time, but I personally don't like GLSL because it requires each shader to occupy a different file.
For a lot of cases, I wish I could write multiple shader entry functions in the same file as they
are closely related. This problem becomes worse when writing ray tracing applications, which usually
involves 4 to 5 shaders.

HLSL is another option. It perfectly resolves the issue that I dislike about GLSL. More importantly,
HLSL is currently adopted by a lot of the game engines, making it a more preferable option over GLSL.
In addition, HLSL also includes Vulkan semantics so that it could be used for Vulkan as well. However,
there are still some issues with HLSL, as I have seen ugly HLSL shaders with both Vulkan semantics
`[[vk::binding(M, N)` and DXIL semantics `regisiter(tM, spaceN)`. Having to manually manage two sets
of annotations is unfortunate.

As the final choice, here's `slang`, which is gaining popularity in the recent years. nVidia has also
adopted slang for their development/reserach engine [Falcor](https://github.com/NVIDIAGameWorks/Falcor).
Slang provides constructs like `ParameterBlock` and uses deterministic algorithm to pick slots for bindings,
so users could avoid manually writing the annotations (although slang also supports the manual annotations).

Additionally, slang has a powerful reflection API that can be used to retrieve basically everything from
the shader. For example, slang supports user attributes, so I can use a `[dynamic]` attribute to denote
a uniform is dynamic.
