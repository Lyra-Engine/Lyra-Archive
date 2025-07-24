+++
date = '2025-07-20T11:00:00-05:00'
draft = false
title = 'RHI: Vulkan'

categories = ["Blog"]
tags = ["RHI", "Vulkan"]

mermaid = true
comments = true
showLicense = true
showRelated = true
+++

## Implementation

The Vulkan implementation has been an easy one, as I can see the WebGPU API
design is largely based on Vulkan's API. Therefore a lot of the API implementation
becomes direct translation. Here's a concept mapping between WebGPU and Vulkan.

## Mapping

| WebGPU             | Vulkan                        |
|--------------------|-------------------------------|
| GPUAdapter         | VkPhysicalDevice              |
| GPUSurface         | VkSurfaceKHR + VkSwapchainKHR |
| GPUDevice          | VkDevice                      |
| GPUCommandBuffer   | VkCommandBuffer               |
| GPURenderPipeline  | VkPipeline                    |
| GPUComputePipeline | VkPipeline                    |
| GPUPipelineLayout  | VkPipelineLayout              |
| GPUBindGroupLayout | VkDescriptorSetLayout         |
| GPUBindGroup       | VkDescriptorSet               |
| GPUBuffer          | VkBuffer                      |
| GPUSampler         | VkSampler                     |
| GPUTexture         | VkImage                       |
| GPUTextureView     | VkImageView                   |
| GPUShaderModule    | VkShaderModule                |
| GPUFence           | VkFence + VkSemaphore         |
| GPUQuerySet        | VkQueryPool                   |

Noteworthily, GPUFence can be used on CPU side, while VkSemaphore is supposed to work
only for GPU/GPU synchronization. However, Vulkan 1.2 introduced timeline semaphore
which can be used on both CPU and GPU. Therefore timeline semaphore is used in most
of the places (except for swapchain, beceause swapchain only works with binary semaphore).

## Layout

WebGPU and Vulkan has exactly the same design for descriptor set.
Both adopts a set-based design, where each set could contain multuple descriptors.
D3D12's design is a little more flexible. This will be covered in another post.

## Memory Management

I have delegated the memory management to GPUOpen's [VulkanMemoryAllocation](https://gpuopen.com/vulkan-memory-allocator/).
With this library I don't have to think about memory sub-allocation, especially when
this part has little to with rendering.

However, there's a drawback. This library will pre-allocate large chunks of memory,
therefore even for the smallest application, it has a minimum memory allocation of about
350Mb. This makes the application somewhat heavy, but since we are aiming at creating game
engine, this is not a problem.

## Descriptor Management

The Vulkan backend adopts a frame-based resource management mechanism. Descriptors
will be only be good for the current frame. Each frame maintains descriptor pools,
and the pools will be reset before the frame starts.
