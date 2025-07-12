# Bevy notes

## Bevy Rendering (WIP)
Bevy as a whole is extremely flexible, where all (or nearly so) aspects of bevy can be swapped out by changing the plugins used. As a result, bevy rendering can also be fully swapped out for a custom renderer. Even within the default bevy rendering plugin, there are usually around multiple methods of doing a specific task, and the preferred method among them evolves over time.

This won't be a comprehensive discussion of bevy rendering, but it will instead be a deeper look at some of the specifics of how bevy rendering interacts with wgpu, particularly in terms of GPU resources like buffers and shaders.


### WGPU + Bevy
Bevy is built on top of a rust library called wgpu, which implements cross-platform rendering allowing a single api to render to WebGPU, Vulkan, Metal, etc. It may lose a very small amount of potential for performance optimizations, but it's not possible to hit that anyway without a team large enough to do the sort of custom rendering that bevy allows anyway by (for example) swapping out the rendering plugin.

By relying on wgpu, bevy can seamlessly support rendering to a huge range of platforms and uses a rendering api very similar to wgpu at the bottom level. Bevy provides higher level apis for rendering, but I'll cover that later. This section will focus on some of the important building blocks of both wgpu and bevy (and a lot of graphics programming in general). Wgpu is conceptually and stylistically similar in API to WebGPU, so you can often directly translate WebGPU experience.

Useful resources:
- Standard rust docs are available for [wgpu](https://docs.rs/wgpu/latest/wgpu/) and [bevy](https://docs.rs/bevy/latest/bevy/)
- Excellent wgpu [tutorial](https://sotrh.github.io/learn-wgpu/)
- Summary article of [bevy rendering](https://hackmd.io/@bevy/rendering_summary). Deep and mostly up to date
- Bevy [cheatbook](https://bevy-cheatbook.github.io/gpu/intro.html). Covers huge swaths of bevy. The rendering discussion of it is a pretty nice intro and has some useful graphics. It's generally out of date, but a lot of it still applies.

#### Vertex Buffers


#### Instance Buffers
Instancing is a graphics utility to create multiple copies of a single mesh, each with their own properties. It is computationally much cheaper than rendering completely different meshes. For a more comprehensive discussion of instancing in wgpu, check out the [Learn Wgpu tutorial on instancing](https://sotrh.github.io/learn-wgpu/beginner/tutorial7-instancing/).


#### Uniform Buffers
Uniform buffers are continuous groups of data on the GPU (buffers) that are availble to all instances of the shader (uniformly accesible). Uniforms have multiple methods of creation in bevy. I'll roughly go in order from simple to hard.


### Bevy Rendering APIs
In addition to the rendering features inherited from wgpu, bevy implements a number of low, medium, and high level rendering APIs that can be used to fully control bevy rendering behavior without diving all the way to WGPU.

#### AsBindGroup
``` rust
    #[derive(AsBindGroup)]
    struct BasicUniform {
        #[uniform(0)]
        a: f32,
    }
```
`AsBindGroup` is a derive macro provided by Bevy which creates all the associated resources (inits the buffer on the GPU, creates the render world resource, and creates the app world resource), and then binds it to the specified bind group number.

#### ExtractComponentPlugin and ExtractUniformPlugin
The two extract plugins (TODO double check if there are others) tell bevy to copy data from the main world to the render world as part of the existing rendering process. This allows that data to be used in the GPU for various resources, with most examples I've seen of the ExtractComponentPlugin for adding data to a vertex buffer (often for instancing or pipeline specialization) and the ExtractUniformPlugin for adding data to a uniform buffer.
 
A useful implementation of the ExtractUniformPlugin in the bevy code is in the [implementation of contrast adaptive sharpening](https://docs.rs/bevy_core_pipeline/0.16.1/src/bevy_core_pipeline/contrast_adaptive_sharpening/mod.rs.html#116) in the bevy core pipeline.

#### Specialized Pipelines (mesh/compute)
Specialization is a set of traits and plugins provided by bevy which allow you to modify the existing pipelines. A common way to use a specialization is to alter an existing mesh pipeline to change the data to a vector buffer. For example, in the [bevy mesh specialization example](https://github.com/bevyengine/bevy/blob/latest/examples/shader/specialized_mesh_pipeline.rs#L209), you can see how the SpecializedMeshPipeline trait allows full customization of the RenderPipelineDescriptor. That allows you to change the data in the vertex buffers, add new buffers, or use custom logic to define state-dependent rendering.

Pipeline specialization is a more complex and involved method of altering bevy rendering behavior, but it is a useful middle ground between small changes possible above and a full customization of the rendering pipeline using WGPU directly.

#### Instancing in Bevy
Bevy supports and uses instancing in multiple ways beyond the general instancing behavior. The easiest to use is automatic instancing which occurs when using the same mesh and material handle. All entities which share mesh and material handles are then rendered as instances in a single batch, even if they're distinct in terms of game logic. You won't get this benefit if you load from source each time you spawn an entity, so you should store and reuse handles as a general rule. 

If writing a shader which is intended to interact with instances, you can use a component called a mesh tag to encode instance information. That can be retrieved with the `get_tag` function from `bevy_pbr`, like below. This function can be used to interact with other components of that entity if you need, as long as those are copied to the render world (and GPU). You'll probably need to implement the ExtractComponent trait and ExtractComponentPlugin the component to specify how to copy to the render world.

``` wgsl
#import bevy_pbr::mesh_functions

struct Vertex {
    @builtin(instance_index) instance_index: u32,
};

@vertex
fn vertex(vertex: Vertex) -> VertexOutput {
    let tag = mesh_functions::get_tag(vertex.instance_index);
...
}
```

A bevy tutorial demonstrating automatic instancing is [here](https://bevyengine.org/examples/shaders/automatic-instancing/).
