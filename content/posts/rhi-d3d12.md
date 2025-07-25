+++
date = '2025-07-20T12:00:00-05:00'
draft = false
title = 'RHI: D3D12'

categories = ["Blog"]
tags = ["RHI", "D3D12"]

mermaid = true
comments = true
showLicense = true
showRelated = true
+++

## Implementation

I have no prior experience writing D3D12, but my complacency after mastering Vulkan has
made me blindly think I can learn as I implement this backend. In fact, I did manage to
implement this backend, but with AI's help.

## Mapping

| WebGPU             | D3D12                         |
|--------------------|-------------------------------|
| GPUAdapter         | IDXGIAdapter1                 |
| GPUSurface         | IDXGISwapChain3               |
| GPUDevice          | ID3D12Device                  |
| GPUCommandBuffer   | ID3D12GraphicsCommandList     |
| GPURenderPipeline  | ID3D12PipelineState           |
| GPUComputePipeline | ID3D12PipelineState           |
| GPUPipelineLayout  | ID3D12RootSignature           |
| GPUBindGroupLayout | D3D12_ROOT_PARAMETER1         |
| GPUBindGroup       | D3D12_CPU_DESCRIPTOR_HANDLE   |
| GPUBuffer          | ID3D12Resource                |
| GPUSampler         | D3D12_CPU_DESCRIPTOR_HANDLE   |
| GPUTexture         | ID3D12Resource                |
| GPUTextureView     | D3D12_CPU_DESCRIPTOR_HANDLE   |
| GPUShaderModule    | std::uint8_t*                 |
| GPUFence           | ID3D12Fence                   |
| GPUQuerySet        | ID3D12QueryHeap               |

As you might have noticed, the mapping from WebGPU to D3D12 is not as straightforward
as the mapping to Vulkan. Here are some items that are worth mentioning.

## Resources

Unlike WebGPU, D3D12 unifies the buffer/texture resources to be `ID3D12Resource`.
Additionally, for ray tracing acceleration structure is also `ID3D12Resource`.
Personally I prefer the clear distinction between different types of objects.

## Samplers

In D3D12, samplers are regular descriptors. Sampler descriptors are managed
in a separate descriptor heap. Under the abstraction of a `GPUSampler` object,
these sampler descriptors are never reset unless being manually destroyed.

A major difference in sampler between D3D12 and WebGPU/Vulkan is, since samplers
are already descriptors, they can be directly bound during command recording.
We are explicitly not using this feature, because D3D12 requires heap to also
be bound in command buffer. The heaps used by command buffers are pertained to
frames, while the sampler heap used for sampler declaration is owned by `RHI`
object itself. As a common practice, we will allocate a sampler descriptor on
the frame heap, and copy the sampler descriptor over.

## Layout

D3D12 adopts a more flexible layout design. Where the top level root root parameter
can be either descriptor table (similar to GPUBindGroupLayout), root constants,
top level CBV / UAVs, etc. For simplicity, we will map GPUBindGroupLayout to root
descriptor table. Bind group layout entries will be mapped to the same descriptor
table and share the same root parameter (so that they could be bound in a single call).

However, there is an exception for dynamic uniform/storage buffer. As D3D12 does not
have the concept of dynamic uniform/storage buffer, it uses `SetGraphicsRootConstantBufferView`
and `SetGraphicsRootUnorderedAccessView` to directly set the GPU buffer virtual address
in order to emulate this feature. As both methods requires a separate root parameter,
we have to move these parameters out from the descriptor table, and assign individual
root parameter index.

## Memory Management

Similar to Vulkan, I have delegated the memory management to GPUOpen's [D3D12MemoryAllocation](https://gpuopen.com/d3d12-memory-allocator/).
This library has exactly the same pros and cons as the Vulkan alternative. See previous post.

## Descriptor Management

The D3D12 backend also adopts a frame-based resource management mechanism. D3D12 has a
more flexible design on the descriptor management, but also has some limitations. One
major limitation is that sampler is separated from rest of the descriptor types.
Therefore, each frame manages two separate descriptor heaps and will be reset upon
frame starts.

The sampler heap has a limit of 2048, which should be big enough for most of the applications.
The heap for other descriptors has a much larger limit because this heap holds all other resources,
including buffers and textures.

Our command buffer will bind the two frame heaps on the start. Throughout the life time of this
command buffer, we will stick to only these two heaps for the ease of heap management.

## Descriptor Handle

Each descriptor handle represents a group of shader inputs. The group of shader inputs,
according to the bind group layout, could contain buffers, textures, samplers, etc.

Note that I have just mentioned samplers needs to be allocated from a separate heap.
This means we have to record multiple descriptor handles: one for cbv/uav/srv, and the
other for samplers. In addition, because dynamic uniform/storage buffers are handled
separately from rest of the descriptors, we also need to separately record a handle for
them. This means another 4-8 bytes of storage.

For descriptor handles, every byte counts because there will be a ton of descriptors used
in applications. Therefore I only record the offset from the base heap. In total there
are 3 uint offsets recorded (16 bytes). Ideally we want 8 bytes, but this is by far what
I can come up with.
