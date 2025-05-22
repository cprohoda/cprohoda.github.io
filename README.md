# Bevy notes

## Bevy Rendering (WIP)
Bevy as a whole is extremely flexible, where all (or nearly so) aspects of bevy can be swapped out by changing the plugins used. As a result, bevy rendering can also be fully swapped out for a custom renderer. Even within the default bevy rendering plugin, there are usually around multiple methods of doing a specific task, and the preferred method among them evolves over time.

This won't be a comprehensive discussion of bevy rendering, but it will instead be a deeper look at some of the specifics of how bevy rendering interacts with wgpu, particularly in terms of GPU resources like buffers and shaders.

### Uniform Buffers
Uniform buffers are continuous groups of data on the GPU (buffers) that are availble to all instances of the shader (uniformly accesible). Uniforms have multiple methods of creation in bevy. I'll roughly go in order from simple to hard.

#### AsBindGroup
``` rust
    #[derive(AsBindGroup)]
    struct BasicUniform {
        #[uniform(0)]
        a: f32,
    }
```
`AsBindGroup` is a derive macro provided by Bevy which creates all the associated resources (inits the buffer on the GPU, creates the render world resource, and creates the app world resource), and then binds it to the specified bind group number.

#### Specialized Pipelines (mesh/compute)
Specialization is a set of traits and plugins provided by bevy which allow you to modify the existing pipelines. A common way to use a specialization is to alter an existing mesh pipeline to change the data to a vector buffer. For example, in the (bevy mesh specialization example)[https://github.com/bevyengine/bevy/blob/latest/examples/shader/specialized_mesh_pipeline.rs#L209], you can see how the SpecializedMeshPipeline trait allows full customization of the RenderPipelineDescriptor. That allows you to change the data in the vertex buffers, add new buffers, or use custom logic to define state-dependent rendering.

Pipeline specialization is a more complex and involved method of altering bevy rendering behavior, but it is a useful middle ground between small changes possible above and a full customization of the rendering pipeline using WGPU directly.
