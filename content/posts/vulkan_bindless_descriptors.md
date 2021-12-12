+++
title = "Bindless descriptor sets"
publishDate = 2021-12-01T00:00:00+01:00
lastmod = 2021-12-12T10:39:17+01:00
tags = ["vulkan", "graphics"]
draft = false
+++

## What is bindless {#what-is-bindless}

From what I can find online, bindless techniques were first mentioned by Nvidia in 2009.

> Bindless Graphics refers to changes to OpenGL that can enable close to an order of magnitude improvement in the CPU-limitedness of graphics applications. [...]
>
> -- Nvidia Bindless Graphics

The main goal of these OpenGL extensions is to offload some work to the GPU, to improve CPU performance with draw intensive scenes.

One of these extensions is `ARB_bindless_texture`.
It allows to completely bypass one of the main CPU cost in rendering: texture bindings.
Binding resources to a slot in DX11 or OpenGL is a costly operation because the driver had to traverse several internal data-structures to lookup the resource, refcount it, validate it, etc.
With the extension, each texture has an associated 64-bits handle that can be passed in uniform buffers like any other data.
This massively reduce the number of resouce binding and API calls, thus increasing performance.


## Vulkan Implementation {#vulkan-implementation}


### Overview {#overview}

Vulkan has a completely different resource binding model than OpenGL with descriptor sets.
With these, it's possible to replicate an OpenGL bindless model manually.
Im going to first show an example of how use this bindless model, and then give implementations details on both the GLSL and Vulkan side.

One of the core ideas of Vulkan is to have multiple descriptor sets that can be bound at different frequencies.
The intended way to use them is to have a descriptor set containing "frame global" data that gets bound once, a descriptor set containings "shader data" that is common across all invocations and a "draw data" descriptor set that needs to be bound for every draw.

```glsl
// Descriptor set #0 contains data valid for the entire frame
layout(set = 0, binding = 0) uniform GlobalUniform {
    float4x4 camera_view;
    float4x4 camera_projection;
    float4x4 camera_view_inverse;
    float4x4 camera_projection_inverse;
    float4x4 camera_previous_view;
    float4x4 camera_previous_projection;
    float2 render_resolution;
    float2 jitter_offset;
    float delta_t;
    uint frame_count;
} globals;

// Descriptor set #1 contains data that is the same across all invocations
layout(set = 1, binding = 0) uniform ShaderOptions {
    float exposure;
    float alpha;
    int debug_display;
} shader_options;

// Descriptor set #2 contains per-draw data, every draw will sample different textures
layout(set = 2, binding = 0) uniform sampler2D albedo_texture;
```

Unfortunately this approach is still very costly when you have too much draw calls...
And `vkCmdBindDescriptorSets` will show up a lot in the profiler next to the `vkCmdDrawIndexed`.

To bindless, we will bind **everything**.
We will use a separate descriptor set for each `VkDescriptorType`.
Every resource will be bound to the corresponding descriptor set at creation, and unbound when destroyed.

```glsl
// Descriptor set #0 contains data valid for the entire frame
layout(set = 0, binding = 0) uniform GlobalUniform {
    float4x4 camera_view;
    float4x4 camera_projection;
    float4x4 camera_view_inverse;
    float4x4 camera_projection_inverse;
    float4x4 camera_previous_view;
    float4x4 camera_previous_projection;
    float2 render_resolution;
    float2 jitter_offset;
    float delta_t;
    uint frame_count;
} globals;

// Descriptor set #1 contains all the VkImage created with VK_IMAGE_USAGE_SAMPLED_BIT  so far, it is bound once per frame
layout(set = 1, binding = 0) uniform sampler2D global_textures[];

// Descriptor set #2 is the same "per-shader" descriptor as the previous example
layout(set = 2, binding = 0) uniform ShaderOptions {
    float exposure;
    float alpha;
    int debug_display;
    uint albedo_texture_indices[];
} shader_options;

void main()
{
    uint albedo_texture_index = shader_options.albedo_texture_indices[gl_DrawID];
    float4 albedo = texture(global_textures[albedo_texture_index], uv);
}
```

The goal here is to remove all per-draw bindings.
As said above, all images will be bound in a global descriptor set (#1 in the example). The global image descriptor set is only bound once per frame.
To bind a resource to a shader, we just need to give it an index (similar to OpenGL bindless handles) into its global descriptor using push constants or a buffer.

The per-draw data can be moved to a per-shader buffer into an array. This array can be indexed using a push constant or `gl_DrawID` when using indirect draw commands.

Using a "bindless" descriptor for the images and moving all the per-draw datain a array reduces the number of `vkCmdBindDescriptorSets` from something dependent on the number of drawcall to a constant.
I hope this example makes sense to you and gives you a general idea on what bindless is and why it is used.
I am now going to explain how to implement it in GLSL and in Vulkan.


### GLSL {#glsl}


#### Descriptor aliasing {#descriptor-aliasing}

GLSL was originally made for OpenGL, not Vulkan. As such, there are some differences in how descriptor bindings are declared.
For example in Vulkan there is only one descriptor type for storage images but in GLSL you have to specify the format and dimensions of the bounded images.
It's posible to use images of different format thanks to a feature called "descriptor aliasing".
Simply put, you can declare different "layout" for the same binding.

<!--list-separator-->

-  Image descriptors

    One set is dedicated to `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` descriptors.

    ```glsl
    layout(set = 1, binding = 0) uniform sampler2D global_textures[];
    layout(set = 1, binding = 0) uniform usampler2D global_textures_uint[];
    layout(set = 1, binding = 0) uniform sampler3D global_textures_3d[];
    layout(set = 1, binding = 0) uniform usampler3D global_textures_3d_uint[];
    ```

    And another one is dedicated to `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE` descriptors.

    ```glsl
    layout(set = 2, binding = 0, rgba8) uniform image2D global_images_2d_rgba8[];
    layout(set = 2, binding = 0, rgba16f) uniform image2D global_images_2d_rgba16f[];
    layout(set = 2, binding = 0, rgba32f) uniform image2D global_images_2d_rgba32f[];
    layout(set = 2, binding = 0, r32f) uniform image2D global_images_2d_r32f[];
    ```

<!--list-separator-->

-  Buffer descriptors

    You can also alias buffer descriptors.
    I prefer to use only storage buffers in my bindless setup and use a "shader" frequency set containing a dynamic uniform buffer to pass shader options and ressources indices.

    ```glsl
    layout(set = 3, binding = 0) buffer UiVerticesBuffer            { ImGuiVertex vertices[];  } global_buffers_ui_vert[];
    layout(set = 3, binding = 0) buffer InstancesBuffer             { RenderInstance render_instances[]; } global_buffers_instances[];
    layout(set = 3, binding = 0) buffer MeshesBuffer                { RenderMesh render_meshes[]; } global_buffers_meshes[];
    layout(set = 3, binding = 0) buffer SubMeshInstancesBuffer      { SubMeshInstance submesh_instances[]; } global_buffers_submesh_instances[];
    layout(set = 3, binding = 0) buffer SubMeshesBuffer             { SubMesh submeshes[]; } global_buffers_submeshes[];
    layout(set = 3, binding = 0) buffer PositionsBuffer             { float4 positions[]; } global_buffers_positions[];
    layout(set = 3, binding = 0) buffer UvsBuffer                   { float2 uvs[]; } global_buffers_uvs[];
    layout(set = 3, binding = 0) buffer IndicesBuffer               { u32 indices[]; } global_buffers_indices[];
    layout(set = 3, binding = 0) buffer BVHBuffer                   { BVHNode nodes[]; } global_buffers_bvh[];
    layout(set = 3, binding = 0) buffer DrawArgumentsBuffer         { u32 draw_count; DrawIndexedOptions arguments[]; } global_buffers_draw_arguments[];
    layout(set = 3, binding = 0) buffer UintBuffer                  { u32 data[]; } global_buffers_uint[];
    layout(set = 3, binding = 0) buffer MaterialsBuffer             { Material materials[]; } global_buffers_materials[];
    ```


#### Descriptor indexing {#descriptor-indexing}

So far I just assumed that it is possible to create arrays of descriptors and index them as you would index an array in C++.
As always in Vulkan, the set of core operations is quite limited and we have to check for capabilities and features supported by the physical device before using advanced features.

By default, it is only possible to access descriptor arrays with **compile-time** indices.

```glsl
void main()
{
    const uint albedo_texture_index = 4;
    float4 albedo = texture(global_textures[albedo_texture_index], uv);
}
```

You can check for the `VkPhysicalDeviceFeatures::shaderUniformBufferArrayDynamicIndexing`, `VkPhysicalDeviceFeatures::shaderSampledImageArrayDynamicIndexing`, `VkPhysicalDeviceFeatures::shaderStorageBufferArrayDynamicIndexing` and `VkPhysicalDeviceFeatures::shaderStorageImageArrayDynamicIndexing` features to enable dynamic indexing.
It makes it possible to access descriptor arrays with **uniform** (constant across invocations) values.

```glsl
layout(set = 2, binding = 0) uniform ShaderOptions {
    float exposure;
    float alpha;
    int debug_display;
    uint albedo_texture_indices[];
} shader_options;

void main()
{
    uint albedo_texture_index = shader_options.debug_display;
    float4 albedo = texture(global_textures[albedo_texture_index], uv);
}
```

Finally, the most recent devices support fully dynamic indexing with the `VK_EXT_descriptor_indexing` extension.
The extension has been promoted to Vulkan 1.2, that means that if you are on a Vulkan 1.2 device, you can just check for the `VkPhysicalDeviceVulkan12Features::descriptorIndexing` feature.
On the GLSL side of things, you have to require the `GL_EXT_nonuniform_qualifier` extension to use non uniform indices. When accessing a descriptor array with such indices, you have to mark them with `nonuniformEXT`.

```glsl
// for nonuniformEXT
#extension GL_EXT_nonuniform_qualifier : require
// for gl_DrawID
#extension GL_ARB_shader_draw_parameters : require


layout(set = 2, binding = 0) uniform ShaderOptions {
    float exposure;
    float alpha;
    int debug_display;
    uint albedo_texture_indices[];
} shader_options;

void main()
{
    uint albedo_texture_index = shader_options.albedo_texture_indices[gl_DrawID];
    float4 albedo = texture(global_textures[nonuniformEXT(albedo_texture_index)], uv);
}
```


### Vulkan {#vulkan}


#### Creating the descriptor pool {#creating-the-descriptor-pool}

The first thing to do is to create a descriptor pool.
I am using a separate descriptor pool for my bindless sets.

```cpp
std::array pool_sizes{
    VkDescriptorPoolSize{.type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, .descriptorCount = 1024},
    VkDescriptorPoolSize{.type = VK_DESCRIPTOR_TYPE_STORAGE_IMAGE,          .descriptorCount = 1024},
    VkDescriptorPoolSize{.type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,         .descriptorCount = 1024},
};

VkDescriptorPoolCreateInfo pool_info = {.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO};
pool_info.poolSizeCount              = static_cast<u32>(pool_sizes.size());
pool_info.pPoolSizes                 = pool_sizes.data();
pool_info.maxSets                    = 3;

VK_CHECK(vkCreateDescriptorPool(device, &pool_info, nullptr, &descriptor_pool));
```


#### Creating the descriptor sets {#creating-the-descriptor-sets}

Next we allocate a separate descriptor set for each `VkDescriptorType`, because only the final binding in a descriptor set can have a variable size. (<https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK%5FEXT%5Fdescriptor%5Findexing.html>).

I update each set once per frame if needed and then bind them at the start of the frame, so I don't need the various `UPDATE_AFTER_BIND` flags.
I setup each set to have one binding of 1024 descriptors, you can try to use the `VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT` flag to allocate only the amount you need but I don't bother to.
I am also using the `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` flag to allow "holes" in the array.

Here is an example of descriptor set creation:

```cpp
VkDescriptorSetLayoutBinding binding = {.binding         = 0,
                                        .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
                                        .descriptorCount = 1024,
                                        .stageFlags      = VK_SHADER_STAGE_ALL};

VkDescriptorBindingFlags flag = VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT;

VkDescriptorSetLayoutBindingFlagsCreateInfo flag_info = {.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_BINDING_FLAGS_CREATE_INFO};
flag_info.bindingCount  = 1;
flag_info.pBindingFlags = &flag;

VkDescriptorSetLayoutCreateInfo set_layout_info = {.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO};
set_layout_info.pNext                           = &flag_info;
set_layout_info.bindingCount                    = 1;
set_layout_info.pBindings                       = &binding;

VK_CHECK(vkCreateDescriptorSetLayout(device, &set_layout_info, nullptr, &set_layout));

VkDescriptorSetAllocateInfo set_info = {.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO};
set_info.descriptorPool              = descriptor_pool;
set_info.pSetLayouts                 = &set_layout;
set_info.descriptorSetCount          = 1;
VK_CHECK(vkAllocateDescriptorSets(device, &set_info, &descriptor_set));
```


#### Binding resources on creation {#binding-resources-on-creation}

To have the same convenience as the OpenGL bindless setup, it is very important to bind resources automatically to our bindless descriptor sets and have a way to get a handle back.

A simple free-list per set is enough to assign a handle.
When a resource is created you pop the head of the free-list to find the first free slot in the set and use that as a handle.
At deletion, you push the handle of the resource to the free-list.

To actually update the descriptor set, each set also maintains a list of pending bind or unbind.
At the start of a frame each set is updated and bound.


#### Descriptor set compatiblity {#descriptor-set-compatiblity}

To be able to bind a set once in a frame for all shaders, all pipeline layouts have to be **compatible**.

> Two pipeline layouts are defined to be “compatible for push constants” if they were created with identical push constant ranges. Two pipeline layouts are defined to be “compatible for set N” if they were created with identically defined descriptor set layouts for sets zero through N, and if they were created with identical push constant ranges.
>
> When binding a descriptor set (see Descriptor Set Binding) to set number N, if the previously bound descriptor sets for sets zero through N-1 were all bound using compatible pipeline layouts, then performing this binding does not disturb any of the lower numbered sets. If, additionally, the previously bound descriptor set for set N was bound using a pipeline layout compatible for set N, then the bindings in sets numbered greater than N are also not disturbed.
>
> Similarly, when binding a pipeline, the pipeline can correctly access any previously bound descriptor sets which were bound with compatible pipeline layouts, as long as all lower numbered sets were also bound with compatible layouts.
>
> Layout compatibility means that descriptor sets can be bound to a command buffer for use by any pipeline created with a compatible pipeline layout, and without having bound a particular pipeline first. It also means that descriptor sets can remain valid across a pipeline change, and the same resources will be accessible to the newly bound pipeline.
>
> -- <https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#descriptorsets-compatibility>

Basically all pipeline layouts have to be created with the bindless descriptor set layouts.
I am not sure if push constants also have to be compatible in this case.
In my toy engine I use the same push constant range for all pipelines anyway.


## Buffer device address {#buffer-device-address}


### Purpose {#purpose}

The setup described in this article works well for textures and buffers.

But for buffer specifically there is another extension called `VK_KHR_buffer_device_address`.
It is core in Vulkan 1.2 and can be enabled with the `VkPhysicalDeviceVulkan12Features::bufferDeviceAddress` feature.
Support in debugging tools such as Renderdoc is separated in another feature called `bufferDeviceAddressCaptureReplay`.
It may be faster than the manual bindless approach as handles are supposed to be the memory address directly.


### Vulkan {#vulkan}

Buffer should be created with the `VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT` flag.
If you use Vulkan Memory Allocator you also have to pass the `VMA_ALLOCATOR_CREATE_BUFFER_DEVICE_ADDRESS_BIT` flag when creating the allocator.

Getting the handle is simply calling `vkGetBufferDeviceAddress`:

```cpp
VkBufferDeviceAddressInfo address_info = {.sType = VK_STRUCTURE_TYPE_BUFFER_DEVICE_ADDRESS_INFO};
address_info.buffer = vk_buffer_handle;
uint64_t gpu_address = vkGetBufferDeviceAddress(device, &address_info);
```

And then you can pass the address with uniform or push constants.


### GLSL {#glsl}

The usage in GLSL is a bit verbose because it isn't the same as using C pointers.
You first have to declare a buffer "type" with the `buffer_reference` layout.
Instead of using a `uint64_t` in GLSL you can directly use the type you declared (like sampler2D in OpenGL bindless textures).

```glsl
#extension GL_EXT_buffer_reference : require

// forward declaration
layout(buffer_reference) buffer blockType;

// complete reference type definition
layout(buffer_reference, std430, buffer_reference_align = 16) buffer blockType {
    int x;
    blockType next;
};

// A normal block declaration that includes a reference to blockType
layout(set = 4, binding = 0) buffer rootBlock {
    blockType root;
} r;

void main()
{
    blockType b = r.root;
    // "pointer chasing" through a linked list
    b = b.next.next.next.next.next;
    // ...
    // use b.x;
}
```

There are various GLSL extensions to make working with buffer addresses easier.

-   `GL_EXT_buffer_reference2` adds pointer math support with addresses.
-   `GL_ARB_gpu_shader_int64` and `GL_EXT_shader_explicit_arithmetic_types_int64` allows to use `uint64_t` in GLSL. You can cast addresses to `uint64_t` and cast `uint64_t` to any buffer type.
-   `GL_EXT_buffer_reference_uvec2` allows to do arithmetics on addresses with `uvec2` for devices that don't support `uint64_t`.


### Conclusion {#conclusion}

This extension is great but there are some capturing issues with AMD drivers.
On Windows the capturing feature is enabled but the driver crashes when using Renderdoc.
On Linux the Mesa driver doesn't implement buffer device addresses.
So I am still using the manual bindless setup for buffers.


## Conclusion {#conclusion}

Bindless is great.
It gives so much flexibility that binding resources is no longer a headache on Vulkan and DX12.
You basically only have to provide an API for sending uniform buffers to your shaders and a way to get handles for textures and buffers.

It also improves CPU performance a lot because the number of api calls is reduced.
There is some overhead on the GPU side because accessing a resource is more complex.
But this is largely compensed because it is possible to batch almost everything and go full GPU-driven.

It is almost required for ray tracing, because you don't know in advance what material a ray will hit.
And the Shader Binding Table is a pain to use without bindless.


## Sources {#sources}

Nvidia Bindless Graphics (2009)
: <https://www.nvidia.com/en-us/drivers/bindless-graphics/>

Vulkan textures unbound
: <https://roar11.com/2019/06/vulkan-textures-unbound/>

New game changing Vulkan extensions for mobile: Descriptor Indexing
: <https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/vulkan-descriptor-indexing>


## Good reads {#good-reads}

"Rendering the hellscape of Doom Eternal" SIGGRAPH 2020
: A good overview of optimisations using bindless textures and unified vertex memory <https://advances.realtimerendering.com/s2020/RenderingDoomEternal.pdf>

"GPU-Driven Rendering Pipelines" SIGGRAPH 2015
: <https://advances.realtimerendering.com/s2015/aaltonenhaar%5Fsiggraph2015%5Fcombined%5Ffinal%5Ffooter%5F220dpi.pdf>