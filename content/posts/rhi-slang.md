+++
date = '2025-07-21T09:00:00-05:00'
draft = false
title = 'RHI: Slang'

categories = ["Blog"]
tags = ["RHI", "Shader", "Slang"]

mermaid = true
comments = true
showLicense = true
showRelated = true
+++

## Why Slang?

As mentioned in the previous post, I chose `shader-slang` for multiple reasons:

1. powerful language constructs
2. powerful reflection api
3. implicit and determinstic binding assign order

Slang has enabled good software programming for shaders. It supports the notion
shader modules. Inside the same slang session, each module will be compiled
exactly once, and linked to other modules as needed.

Slang has good support for generics. The same logic could be applied without
tedious code duplication or preprocessor tricks. From a language design point
of view, it is a much cleaner and modern design than GLSL/HLSL.

Slang has introduced `ParameterBlock`, which naturally organizes shader inputs
based on bind group / descriptor set. All shader bindings are assigned based on
a simple incremental order, which is the most natural thing people will do.
In fact, WebGPU does not support non-consecutive bind groups either. Therefore
purely using `ParameterBlock` suffices the need of our RHI API.

Although slang supports regular `[[vk::binding]]` and `register()` annotations,
the official documentation recommends using `ParameterBlock` over manual annotations.

## API Design

Following is my tentative API design for shader compiler, including both compilation
and reflection:

```cpp
struct ShaderAPI
{
    // backend api name for shader compiler
    CString (*get_api_name)();

    bool (*create_compiler)(CompilerHandle& compiler, const CompilerDescriptor& descriptor);
    void (*delete_compiler)(CompilerHandle compiler);

    // compile from source
    bool (*create_module)(CompilerHandle compiler, const CompileDescriptor& desc, ShaderModuleHandle& module);
    void (*delete_module)(ShaderModuleHandle module);

    // reflect from modules
    bool (*create_reflection)(CompilerHandle compiler, ShaderEntryPoints entries, ShaderReflectionHandle& reflection);
    void (*delete_reflection)(ShaderReflectionHandle reflection);

    // retrieve shader blobs
    bool (*get_shader_blob)(ShaderEntryPoint entry, ShaderBlob& blob);
    bool (*get_vertex_attributes)(ShaderReflectionHandle reflection, ShaderAttributes attrs, GPUVertexAttribute* attributes);
    bool (*get_bind_group_layouts)(ShaderReflectionHandle reflection, uint& count, GPUBindGroupLayoutDescriptor* layouts);
    bool (*get_bind_group_location)(ShaderReflectionHandle reflection, CString name, uint& location);
};
```

## Compilation

The shader compilation process involves compiler creation and module compilation.
The compiler is used thought the application in order to share the sessions between
modules.

```cpp
auto compiler = execute([&]() {
    auto desc   = CompilerDescriptor{};
    desc.target = CompileTarget::SPIRV;
    desc.flags  = CompileFlag::DEBUG;
    return Compiler::init(desc);
});

auto module = execute([&]() {
    auto desc   = CompileDescriptor{};
    desc.module = "test";
    desc.path   = "test.slang";
    desc.source = program_source;
    return compiler->compile(desc);
});

// spirv binary
auto code  = module->get_shader_blob("vsmain");
```

## Reflection

In order to have a complete view of all the shaders used within a single pipeline.
I can ask compiler to create a reflection object using all of the specified entries.
For example:

```cpp
auto reflection = compiler->reflect({
    {*module, "vsmain"},
    {*module, "fsmain"},
});
```

This reflection object can be used to retrieve
1. vertex attributes
2. bind group layout

### Vertex Attributes

Vulkan and D3D12 uses different style of vertex attribute annotation.
Both Vulkan and WebGPU use single attribute location to denote a vertex attribute,
while D3D12 uses a semantic name and a semantic index to denote a vertex attribute.

```cpp
struct VertexInput
{
    float3 position : POSITION;     // semantic name: POSITION, semantic index: 0, location: 0
    float2 texcoord : TEXCOORD0;    // semantic name: TEXCOORD, semantic index: 0, location: 1
    float3 color    : COLOR0;       // semantic name: COLOR,    semantic index: 0, location: 2
};
```

During vertex buffer layout creation in pipeline, D3D12 requires explicit semantic name
and index. Note that this semantic index is different from absolute attribute location
that other APIs use. In order not to manually manage two sets of locations, using the
reflection API is the best choice here.

Since shader won't have the vertex buffer layout information, what we can do is only
to ask reflection to fill out the shader location / shader semantic for us.
The offset of a shader vertex attribute still have to be provided manually.

Here's an example to reflect a set of vertex inputs (used by a single vertex buffer layout).
For vertex buffer layout creation, user would still have to provide stride and step mode.

```cpp
auto attribs = reflection->get_vertex_attributes({
    {"position", offsetof(Vertex, position)},
    {"color",    offsetof(Vertex, color)   },
});
```

### Bind Group Layout

One of the most natural thing for shader reflection to do is the bind group layout reflection,
as this is usually the most tedious part, where user has to define the layout both in shader
and host side RHI. Lyra-Engine provides an API to get all the bind group layouts used in a shader.

```cpp
for (auto& desc : reflection->get_bind_group_layouts()) {
    auto blayout = device.create_bind_group_layout(desc);
    ...
}
```

There are some bits of information that cannot be easily reflected from shaders. For example,
dynamic uniform/storage buffer is no different in the shader. For these cases, Lyra-Engine will
resolve it using slang's custom attributes (which is another reason for picking slang):

```cpp
import lyra;

struct MVP
{
    float4x4 xform;
};

[lyra::dynamic]
ConstantBuffer<MVP> mvp;
```

## Last Words

Slang's reflection API is the hardest shader reflection API among the ones I have used.
Shader reflection APIs like SPIRV-reflect allows users to directly iterate through the
sets / bindings. This reflection API is output-oriented.

Slang's reflection API is different. It is input-oriented. Its reflection API is defined
in a way that asks users to traverse the abstract syntax tree (AST), while the documentation
for the AST is not so well written. Therefore it took some amount of trial and error, combined
with guessing to reach out the final implementation.

I cannot say my implementation is impeccable, but it is good for now. We will fix new problems
as we write more applications in the future.

## References

- [Compilation API](https://shader-slang.org/docs/compilation-api/)
- [Reflection API](https://shader-slang.org/slang/user-guide/reflection)
