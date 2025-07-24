+++
date = '2025-07-20T10:00:00-05:00'
draft = false
title = 'RHI Overview'

categories = ["Blog"]
tags = ["Design", "RHI"]

mermaid = true
comments = true
showLicense = true
showRelated = true
+++

## WebGPU API

As the previous post has mentioned, I have chosen to use
[WebGPU](https://www.w3.org/TR/webgpu/) API as the foundation for my RHI
abstraction layer. In this post, I will talk about the deviations from this
foundation API.

## Explicit Lifetime Management

WebGPU automatically tracks the lifetime of all GPU objects. This is one of the key
points WebGPU is simpler to use. It is similar to Metal 1.0 in a way. However, the
trend of modern rendering is that graphics API reduces overhead by asking users to
manage their own stuff. Therefore I choose to let users create/destroy GPU objects
manually.

## Explicit Synchronization

I added the explicit synchronization control back into the API.
The synchronization types: `GPUBarrierSync`, `GPUBarrierAccess` and `GPUBarrierLayout`
are directly copied from [D3D12 Enhanced Barriers](https://microsoft.github.io/DirectX-Specs/d3d/D3D12EnhancedBarriers.html).
The major reason for adding these back was to make the driver part easier, and to allow
users to perform application level optimization.

## Missing Features

There are a few constructs missing from WebGPU.

1. push constants
2. bindless samplers/textures
3. ray tracing support

### Push Constants

Push constant is a relatively fundamental feature. It is not hard to find the equivalent
support across Vulkan, D3D12 and Metal. I am not sure why WebGPU has decided to not include
it in the first place. There has been a github [issue](https://github.com/gpuweb/gpuweb/issues/75)
on supporting push constants since 2018, but it is still open. However, we can take a look at
how other WebGPU ports (for example, [wgpu](https://github.com/gfx-rs/wgpu)) are designing the API.

### Bindless

Bindless is a key feature in modern rendering. GPU driven rendering pipelines are built on top
of bindless textures. In the era of ray tracing, this is even more true because ray tracing
requires all resources to be available to the shader.

There is a github [issue](https://github.com/gfx-rs/wgpu/issues/3637) on wgpu about bindless support,
but it has not landed anywhere yet. For this we might just have to come up with a design myself.
This is worth another post.

### Ray Tracing

Ray tracing is a must have feature since hardware ray-tracing support is becoming mature.
With the help of ray tracing, many old technique could simple retire. In an ideal world,
we no longer need to fake global illumination and mirror reflection. Shadows will have
no aliasing due to small shadow resolution.

This is similar to the bindless case that no other WebGPU implementations have done so.
wgpu has a github [issue](https://github.com/gfx-rs/wgpu/issues/6762), but it is not materialized.

## API Design

Based on the issues described above, I have tentatively come up with this set of API:

```cpp
struct RenderAPI
{
    // api name
    CString (*get_api_name)();

    bool (*create_instance)(const RHIDescriptor& descriptor);
    void (*delete_instance)();

    bool (*create_adapter)(GPUAdapter& adapter, const GPUAdapterDescriptor& descriptor);
    void (*delete_adapter)();

    bool (*create_surface)(GPUSurfaceHandle& surface, const GPUSurfaceDescriptor& descriptor);
    void (*delete_surface)(GPUSurfaceHandle surface);
    bool (*get_surface_extent)(GPUSurfaceHandle surface, GPUExtent2D& extent);
    bool (*get_surface_format)(GPUSurfaceHandle surface, GPUTextureFormat& format);

    bool (*create_device)(const GPUDeviceDescriptor& descriptor);
    void (*delete_device)();

    bool (*create_fence)(GPUFenceHandle& fence);
    void (*delete_fence)(GPUFenceHandle fence);

    bool (*create_buffer)(GPUBufferHandle& buffer, const GPUBufferDescriptor& descriptor);
    void (*delete_buffer)(GPUBufferHandle buffer);

    bool (*create_sampler)(GPUSamplerHandle& sampler, const GPUSamplerDescriptor& descriptor);
    void (*delete_sampler)(GPUSamplerHandle sampler);

    bool (*create_texture)(GPUTextureHandle& texture, const GPUTextureDescriptor& descriptor);
    void (*delete_texture)(GPUTextureHandle texture);
    bool (*create_texture_view)(GPUTextureViewHandle& view, GPUTextureHandle texture, const GPUTextureViewDescriptor& descriptor);
    void (*delete_texture_view)(GPUTextureViewHandle view);

    bool (*create_shader_module)(GPUShaderModuleHandle& texture, const GPUShaderModuleDescriptor& descriptor);
    void (*delete_shader_module)(GPUShaderModuleHandle texture);

    bool (*create_query_set)(GPUQuerySetHandle& query, const GPUQuerySetDescriptor& descriptor);
    void (*delete_query_set)(GPUQuerySetHandle query);

    bool (*create_blas)(GPUBlasHandle& blas, const GPUBlasDescriptor& descriptor, GPUBlasGeometrySizeDescriptors sizes);
    void (*delete_blas)(GPUBlasHandle blas);

    bool (*create_tlas)(GPUTlasHandle& tlas, const GPUTlasDescriptor& descriptor);
    void (*delete_tlas)(GPUTlasHandle tlas);

    bool (*create_pipeline_layout)(GPUPipelineLayoutHandle& layout, const GPUPipelineLayoutDescriptor& descriptor);
    void (*delete_pipeline_layout)(GPUPipelineLayoutHandle layout);

    bool (*create_render_pipeline)(GPURenderPipelineHandle& texture, const GPURenderPipelineDescriptor& descriptor);
    void (*delete_render_pipeline)(GPURenderPipelineHandle texture);

    bool (*create_compute_pipeline)(GPUComputePipelineHandle& texture, const GPUComputePipelineDescriptor& descriptor);
    void (*delete_compute_pipeline)(GPUComputePipelineHandle texture);

    bool (*create_raytracing_pipeline)(GPURayTracingPipelineHandle& texture, const GPURayTracingPipelineDescriptor& descriptor);
    void (*delete_raytracing_pipeline)(GPURayTracingPipelineHandle texture);

    bool (*create_bind_group)(GPUBindGroupHandle& layout, const GPUBindGroupDescriptor& descriptor);
    bool (*create_bind_group_layout)(GPUBindGroupLayoutHandle& layout, const GPUBindGroupLayoutDescriptor& descriptor);
    void (*delete_bind_group_layout)(GPUBindGroupLayoutHandle layout);

    void (*new_frame)();
    void (*end_frame)();

    bool (*acquire_next_frame)(GPUSurfaceHandle surface, GPUTextureHandle& texture, GPUTextureViewHandle& view, GPUFenceHandle& image_available, GPUFenceHandle& render_complete, bool& suboptimal);
    bool (*present_curr_frame)(GPUSurfaceHandle surface);

    void (*get_mapped_range)(GPUBufferHandle buffer, MappedBufferRange& range);
    void (*map_buffer)(GPUBufferHandle buffer, GPUMapMode mode, GPUSize64 offset, GPUSize64 size);
    void (*unmap_buffer)(GPUBufferHandle buffer);

    void (*wait_idle)();
    void (*wait_fence)(GPUFenceHandle fence);

    bool (*get_blas_sizes)(GPUBlasHandle blas, GPUBVHSizes& sizes);
    bool (*get_tlas_sizes)(GPUTlasHandle tlas, GPUBVHSizes& sizes);

    bool (*create_command_buffer)(GPUCommandEncoderHandle& cmdbuffer, const GPUCommandBufferDescriptor& descriptor);
    bool (*create_command_bundle)(GPUCommandEncoderHandle& cmdbuffer, const GPUCommandBundleDescriptor& descriptor);
    bool (*submit_command_buffer)(GPUCommandEncoderHandle cmdbuffer);

    void (*cmd_wait_fence)(GPUCommandEncoderHandle cmdbuffer, GPUFenceHandle fence, GPUBarrierSyncFlags sync);
    void (*cmd_signal_fence)(GPUCommandEncoderHandle cmdbuffer, GPUFenceHandle fence, GPUBarrierSyncFlags sync);
    void (*cmd_begin_render_pass)(GPUCommandEncoderHandle cmdbuffer, const GPURenderPassDescriptor& descriptor);
    void (*cmd_end_render_pass)(GPUCommandEncoderHandle cmdbuffer);
    void (*cmd_set_render_pipeline)(GPUCommandEncoderHandle cmdbuffer, GPURenderPipelineHandle pipeline);
    void (*cmd_set_compute_pipeline)(GPUCommandEncoderHandle cmdbuffer, GPUComputePipelineHandle pipeline);
    void (*cmd_set_raytracing_pipeline)(GPUCommandEncoderHandle cmdbuffer, GPURayTracingPipelineHandle pipeline);
    void (*cmd_set_bind_group)(GPUCommandEncoderHandle cmdbuffer, GPUIndex32 index, GPUBindGroupHandle bind_group, GPUBufferDynamicOffsets dynamic_offsets);
    void (*cmd_set_index_buffer)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle buffer, GPUIndexFormat format, GPUSize64 offset, GPUSize64 size);
    void (*cmd_set_vertex_buffer)(GPUCommandEncoderHandle cmdbuffer, GPUIndex32 slot, GPUBufferHandle buffer, GPUSize64 offset, GPUSize64 size);
    void (*cmd_draw)(GPUCommandEncoderHandle cmdbuffer, GPUSize32 vertex_count, GPUSize32 instance_count, GPUSize32 first_vertex, GPUSize32 first_instance);
    void (*cmd_draw_indexed)(GPUCommandEncoderHandle cmdbuffer, GPUSize32 index_count, GPUSize32 instance_count, GPUSize32 first_index, GPUSignedOffset32 base_vertex, GPUSize32 first_instance);
    void (*cmd_draw_indirect)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle indirect_buffer, GPUSize64 indirect_offset, GPUSize32 draw_count);
    void (*cmd_draw_indexed_indirect)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle indirect_buffer, GPUSize64 indirect_offset, GPUSize32 draw_count);
    void (*cmd_dispatch_workgroups)(GPUCommandEncoderHandle cmdbuffer, GPUSize32 x, GPUSize32 y, GPUSize32 z);
    void (*cmd_dispatch_workgroups_indirect)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle indirect_buffer, GPUSize64 indirect_offset);
    void (*cmd_copy_buffer_to_buffer)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle source, GPUSize64 source_offset, GPUBufferHandle destination, GPUSize64 destination_offset, GPUSize64 size);
    void (*cmd_copy_buffer_to_texture)(GPUCommandEncoderHandle cmdbuffer, const GPUTexelCopyBufferInfo& source, const GPUTexelCopyTextureInfo& destination, GPUExtent3D copy_size);
    void (*cmd_copy_texture_to_buffer)(GPUCommandEncoderHandle cmdbuffer, const GPUTexelCopyTextureInfo& source, const GPUTexelCopyBufferInfo& destination, const GPUExtent3D& copy_size);
    void (*cmd_copy_texture_to_texture)(GPUCommandEncoderHandle cmdbuffer, const GPUTexelCopyTextureInfo& source, const GPUTexelCopyTextureInfo& destination, const GPUExtent3D& copy_size);
    void (*cmd_clear_buffer)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle buffer, GPUSize64 offset, GPUSize64 size);
    void (*cmd_clear_texture)(GPUCommandEncoderHandle cmdbuffer, GPUTextureHandle texture, const GPUTextureSubresourceRange& range);
    void (*cmd_set_viewport)(GPUCommandEncoderHandle cmdbuffer, float x, float y, float w, float h, float min_depth, float max_depth);
    void (*cmd_set_scissor_rect)(GPUCommandEncoderHandle cmdbuffer, GPUIntegerCoordinate x, GPUIntegerCoordinate y, GPUIntegerCoordinate w, GPUIntegerCoordinate h);
    void (*cmd_set_blend_constant)(GPUCommandEncoderHandle cmdbuffer, GPUColor color);
    void (*cmd_set_stencil_reference)(GPUCommandEncoderHandle cmdbuffer, GPUStencilValue reference);
    void (*cmd_begin_occlusion_query)(GPUCommandEncoderHandle cmdbuffer, GPUSize32 query_index);
    void (*cmd_end_occlusion_query)(GPUCommandEncoderHandle cmdbuffer);
    void (*cmd_write_timestamp)(GPUCommandEncoderHandle cmdbuffer, GPUQuerySetHandle query_set, GPUSize32 query_index);
    void (*cmd_write_blas_properties)(GPUCommandEncoderHandle cmdbuffer, GPUQuerySetHandle query_set, GPUSize32 query_index, GPUBlasHandle blas);
    void (*cmd_resolve_query_set)(GPUCommandEncoderHandle cmdbuffer, GPUQuerySetHandle query_set, GPUSize32 first_query, GPUSize32 query_count, GPUBufferHandle destination, GPUSize64 destination_offset);
    void (*cmd_memory_barrier)(GPUCommandEncoderHandle cmdbuffer, GPUMemoryBarriers barriers);
    void (*cmd_buffer_barrier)(GPUCommandEncoderHandle cmdbuffer, GPUBufferBarriers barriers);
    void (*cmd_texture_barrier)(GPUCommandEncoderHandle cmdbuffer, GPUTextureBarriers barriers);
    void (*cmd_build_tlases)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle scratch_buffer, GPUTlasBuildEntries entries);
    void (*cmd_build_blases)(GPUCommandEncoderHandle cmdbuffer, GPUBufferHandle scratch_buffer, GPUBlasBuildEntries entries);
    void (*cmd_copy_blas)(GPUCommandEncoderHandle cmdbuffer, GPUBlasHandle old_blas, GPUBlasHandle new_blas);
};
```
