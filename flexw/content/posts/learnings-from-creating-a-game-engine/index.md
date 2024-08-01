---
title: 'Learnings From Creating a Game Engine'
date: 2024-07-28T10:00:00+02:00
draft: true
layout: post
---

## TLDR;

I created a 3D Game Engine from scratch and released a small game prototype with it and I want to share my journey, learnings and the architecture of the engine.

## Motivation

During the last time at University I got bored with my side programming projects that where mostly centered around Linux desktop, system and mobile android applications so I asked myself from which area of computing do I have no clue about how its done. I came to the conclusion that I have no idea how games work and how they draw their worlds. This appeared to me as magic. At this point I had no idea about games. The last time I played a game seriously was probably 10 years before when I was a teenager. I searched online for learning resources that would me allow to draw 3D graphics myself. I ended up on [LearnOpenGL.com](https://learnopengl.com) and got very quickly addicted to learning everything about 3D graphics and game development. I was fascinated by the idea to create my own game engine. 

## General

Creating your own game engine is a big undertaking. From my first attempts until now four years passed. I did not continousley work on the same project. My hard drive is filled up with different attempts and of course sometimes I took breaks from that project.

### Know what you want to build

It sounds obvious, but my first attempts failed for several reasons. A very important reason was that during the first attempts I didn't had a clear goal in mind. I tried to build a general purpose Game Engine with no actual usecase (game) in mind. Game Engines do not have meaning without games that have been created with it. As a beginner I saw the big general purpose engines like Unreal Engine, Unity and I was thinking that this is the way a Game Engine has to look and work. That is of course wrong. Instead of trying to build a general purpose engine try to build a very specific (simple) game and do not think much about how you structure your code. If you don't do that, you waste your time with features that you never need and that probably work pretty bad. Building a game gives you confidence that what you are build actually works. In my first attempts I did spend weeks building a editor but it was not even possible to deploy a game with the engine! Of course the editor had a lot of flaws because I did not had anything to proof it's usefulness.

### Do not think about architecture

When I started my gamedev journey I was very well indoctrinated by Object-Oriented Programming (OOP) and all its design patterns and best practices that come with it. Naturally I searched for patterns and best practices for game engine development. I wasted a lot of time by trying to fit my code into design patterns and SOLID principles. That usually led to a point where my project was very difficult to modify and I couldn't move fast anymore. Sometimes I wouldn't realize features because I didn't knew how they would fit into my engine architecture. Luckily at some point I discovered a different thought of school of programming makes it easier to design applications. Namley procedural programming. My current engine is a very simple old school C++ code base without any classes, virtual functions constructors and destructors or templates. I want to make it clear that I'm not against OOP, but it didn't worked well for me for that project. So if you want to realize a feature do not think about which objects and methods you need to realize it. Instead create a function with some input arguments and potentially output arguments and just code the logic inside the function. That approach works very well for me. If I then realize that I need to group some parameters for convinience I create a new struct for it. Or if I realize I need a new function that can be shared I extract it from my written code. This technique is called [semantic compression](https://caseymuratori.com/blog_0015) From that the engine architecture evolves very naturally. I also forced myself to leave inheritance out of the game which was very difficult for me in the beginning. How can code be extendable without inheritance? There a different ways to get something similar to inheritance in C like unions or functions pointers, but there are very few places in my engine where I make use of that and I have the feeling that not making use of inheritance makes my code easier to read and reason about. With these techniques I spend less time building bad abstractions and spend more time building features.

## Engine Architecture

### Memory Management

As my engine is built on C++ I have the luxuray to manage the memory however I want. The goal for me was to have zero allocations while the game runs. I achieve this with the following:

- On startup I allocate a big chunk of memory. E.g. 2 GB. The engine and game can only use that memory block for all it's allocations. If it needs more than these 2 GB the game crashes.

- I provide two type of memory allocators. A dynamic general purpose allocator and a stack allocator.

Writing a good and fast general purpose allocator is difficult. I decided I leave this excerise for another day and use the small [tlsf library](http://www.gii.upv.es/tlsf/) for that. The engine and game are only allowed to call this allocator on startup and occasionley while running the game to for example resize big arrays. E.g. the sprite renderer keeps a array of all sprites it needs to render during a frame and it may needs to resize that array at somepoint if it doesn't has enough space anymore, but this happens very infrequently.

During a frame the engine and the game need to a lot of small allocations that are only valid for one frame. That is where the stack allocator comes into play. The stack allocator allocates a chunk of memory (e.g. 100 MB) during startup. Then during the frame the engine and game may allocate memory as they see fit with it. After the frame has been drawn the stack allocator resets it's memory and the game and engine can allocate memory with it again. That means the stack allocator never needs to do any dynamic allocation and memory stays only valid for one frame. This as well frees the logic during gameplay to think about freeing memory allocations.

This approach for memory management worked quite well for me. It also makes it not necessary anymore to use C++ smart pointers.

In addition, my general purpose allocator has a debug mode that allows me to track memory leaks with it.

### Virtual File System

Originally I wanted to build a game for Android. After I had a first prototype running (with gestures) I realized that I'm not interessed in mobile games. But the Android environment brings in the challenge to think about how you want to access your asset files during runtime. This lead me to a virtual file system that makes it possible to emulate a file system even if it's not physically there. For example my engine can read assets from disk, from a zip file, and even from a zip file that purley exists in memory. That brings the benefit that I can ship my game as either a single executable with a zip file that contains all assets or even just a single executable that has the zip file embedeed. More on that later.

In my oppinion a virtual file system is something that every engine should have right from the beginning to make life easier later. I used the implementation from [this book](https://www.packtpub.com/en-us/product/mastering-android-ndk-9781785288333?srsltid=AfmBOoowzeZNAmIrkyW7Dp-eKAOlACeCDgnp45Lckrk6qf0bNmmyFWju) as starting point and inspiration.

### Config System

To quickly tune some parameters (even during runtime) for the engine and game I have a simple config variable system setup. The system consists of a textfile in the [ini format](https://en.wikipedia.org/wiki/INI_file) where parameters can be tweaked. I have a filewatcher running during runtime that watches the the config file for modifications and if it has been modified it will update the variables during runtime. This system is very efficient and comes with no overhead from just hardcoded variables. For this I was inspired by [Quakes cvar system](https://github.com/id-Software/Quake/blob/master/WinQuake/cvar.h).

### Graphics

My renderer is a Clustered Deferred Renderer built on top of OpenGL. Here is a overview of one frame captured with RenderDoc.

![Frame overview](images/frame_overview.png)

Why did I use OpenGL? First I'm very familiar with the API can work with it quickly. I think OpenGL is high level enough to not care about some specifics that are not interessting to me (yet) and secondly it provide enough low level access to control the things I care about. Overall, I think it's a great API.

The decision for a Clustered Deferred Renderer was driven by mainly two thoughts: I wanted to be able to render many lights and I wanted to have a proofen architecture. Clustered Deferred Rendering is what most engines do these days so I do the same.

#### Frame Breakdown

![Frame](images/frame.png)

Lets take a look on how this frame gets composed.

##### GBuffer

This stage collects draws all meshes and outputs a GBuffer. The Gbuffer contains all the information that is required to shade the pixels later. The Gbuffer contains the albedo, emissive, normals, metallic, roughness, occlusion, position, and shading model. I use the following format for it

```c
B8G8R8A8_UNORM  // albedo + shading model
B8G8R8_UNORM    // emissive
R16G16_FLOAT    // normals
B8G8R8_UNORM    // metallic + roughness + occlusion
R32G32B32_FLOAT // position
```

I compress the normals with a technique called [Octahedral Encoding](https://twitter.com/Stubbesaurus/status/937994790553227264?s=20&t=U36PKMj7v2BFeQwDX6gEGQ). My format is far from optimal, but it gets the job done.

Here are screenshots from the individual outputs.

Albedo with shading model buffer

![Albedo buffer](images/albedo_buffer.png)

Emissive buffer

![Emissive buffer](images/emissive_buffer.png)

Normals buffer

![Normals buffer](images/normals_buffer.png)

Metallic Roughness Occlusions buffer

![Metallic Roughness Occlusions buffer](images/metallic_roughness_occlusion_buffer.png)

Positions buffer

![Position buffer](images/positions_buffer.png)

In addition, this stages outputs a depth buffer

##### Lighting

After the gbuffer was generated it gets processed by the lighting stage. My renderer can support thousands of point lights. To iterate over all thousand lights would be impossible when trying to achieve a good frame rate. I divide the view frustum into a cluster. The screen gets divided into tiles. A tile consists of a 2d bounding box and a depth range. During the light assignment pass each tile gets assigned the lights that affect a given tile. This screenshot visualizes the tiles. Note that the sky an UI doesn't get tiled because no lightning affects it.

![Tiles](images/tiles.png)

Here is a screenshot that visualizes which of the tiles are affected by the two point lights from the rockets. The tiles that get affected by a point light are marked yellow.

![Light tiles](images/light_tiles.png)

The lights get then used during shading to compute each pixels color. For shading I use a Physically-Based BRDF. To get proper ambient light I use a poor mans Global Illumination technique called Image-Based lightning (IBL). IBL computes the ambient lighting based on a environment map. The PBR and IBL implementation is based on the implementation of [3D Graphics Rendering Cookbook](https://www.packtpub.com/en-us/product/3d-graphics-rendering-cookbook-9781838986193?srsltid=AfmBOorSHD5kWqXPtNFlIlflOypMS4R8b_tqdUJOC8sC1CJzvYYFA7ut). The Clustered Deferred Rendering architecture is inspired by the book [Mastering Graphics Programming with Vulkan](https://www.packtpub.com/en-us/product/mastering-graphics-programming-with-vulkan-9781803244792?srsltid=AfmBOopM2E-AGT47Cij05zmSeLGboryDIzC9gDDdT7MIXwHnaTnynoyL).

##### Transparent

After the lighting is done, a transparent forward pass gets executed. The transparent objects gets sorted from back to front and then get drawn with a disabled depth test. Handling of transparent objects is very simple right now. Transparent objects do not get shaded at all.

##### Bloom

To make bright colors appear really bright a physical based Bloom stage gets executed. The bloom stage consists of several passes where first a bright portion from the lightning texture gets isolated and then downsampled over several passes and then upsampled again. This upsampled texture gets then blended on top of the lightning texture during tone mapping. The implementation is inspired by this [video](https://www.youtube.com/watch?v=tI70-HIc5ro) and the presentation by from [Call of Duty](https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/).

This is a picture of the final upsampled bloom texture.

![Bloom](images/bloom.png)

##### Tone Mapping

The lightning and bloom stage put out color textures that contain color values that go above (1, 1, 1). Simply merging the bloom and lightning texture and displaying it would look not very pleasent. In bright areas everything would look white because computer displays have a limited color range and usually show color values above (1, 1, 1) as white. Tone mapping allows to map the colors into a 0 - 1 range that then can be displayed by a computer. I use a ACES tonemapping algorithm. Applying simple static tonemapping is not enough though. For very bright environments other values need to be chosen than for very dark environments. To account for that I use an light adaption algorithm.

The light adaption algorithm first extracts the brightness of the image by downsampling the lightning texture to a single pixel. To get a smooth transition when changing from bright into dark environments and vice versa a adaption algorithm gets used to continously adapt the values over time for the final ACES tone mapping.

My implementation was inspired by the implementation in [3D Graphics Rendering Cookbook](https://www.packtpub.com/en-us/product/3d-graphics-rendering-cookbook-9781838986193?srsltid=AfmBOorSHD5kWqXPtNFlIlflOypMS4R8b_tqdUJOC8sC1CJzvYYFA7ut).

##### UI, Debug

Finally the UI and debug drawing gets drawn above everything else.

This completes a frame in my renderer.

#### Draw Call Submission

A big problem with OpenGL is that its a big state machine. Meaning if you change some state in it every subsequent calls will be affected by the changes as well. This makes it especially difficult to reason about it in bigger projects. Another downside is that OpenGL functions can only be invoked from a single thread. To work around it created a rendering API on top of it that works almost stateless and in a way mimics Vulkans API.

The rendering API allows to create rendering resources like buffers, textures, resource lists, pipelines and command buffers.

During a frame all rendering commands get recorded in one or multiple command buffers (its possible to record them in parallel). This command buffer then gets submitted to the rendering backend at the end of a frame. The rendering backend then processes the commands and translates them into OpenGL commands and executes them. This approach allows to perform optimizations on the order of draw calls and in addition to parallelize the submission of draw calls and it allows running the OpenGL backend in a separate thread. Currently I do none of these optimizations because there is simply no need for it.

```c
dc_cmd_buffer* cmd_buffer = dc_gpu_device_command_buffer(gpu_device);
dc_cmd_buffer_set_index_buffer(cmd_buffer, index_buffer);
dc_cmd_buffer_draw_indexed(cmd_buffer, 0, indices_count);
```

#### Material System

On top of that render abstraction I decided to build a material system. I decided to experiment with a material system that is fully configurable with text files. For this I wrote my own lexer and parsers and ended up with a system that lets me configure shaders, materials, their state and the needed resources for rendering in simple text files. To complement the configurable shader and material system I have also the possibility to configure uniform buffers and storage buffers in a text file. The engine will create these buffers automatically on startup. This reduces potential errors when changing data structures in only in shaders but not on the C++ side. This way there is only single ground truth.

This is how the gbuffer shader looks in my engine with that system. I've augumented the shader with comments to explain the various parts.

```txt
shader gbuffer
{
    // Properties can be set by materials
    // A uniform buffer from the properties will be generated for the shader
    properties
    {
        vec4 u_albedo (0.2, 0.2, 0.2, 1.0)
        vec4 u_emissive (0.0, 0.0, 0.0, 1.0)
        vec4 u_metallic (0.2, 0.2, 0.2, 0.2)
        vec4 u_roughness (0.8, 0.8, 0.8, 0.8)

        texture_2d u_tx_albedo "__default_tex_albedo__" linear_sampler
        texture_2d u_tx_normal "__default_tex_normal__" linear_sampler
        texture_2d u_tx_metallic_roughness "__default_tex_metallic_roughness__" linear_sampler
        texture_2d u_tx_emissive "__default_tex_emissive__" linear_sampler
        texture_2d u_tx_ao "__default_tex_ao__" linear_sampler
    }

    // The layout specifies the resource layout of the shader
    // A list is a collection of resources.
    // One or multiple lists can be bound to a shader.
    // It mostly makes sense to group resources together in a 
    // list that get changed together.
    layout
    {
        list pbr_gbuffer_list
        {
            // A uniform buffer that stores globals that are shared and used by
            // multiple shaders. The actual definition of the uniform buffer is
            // in a different file. It will be shown below.
            uniform_buffer pbr_globals

            // The pbr_instance uniform buffer holds information 
            // that are specific to the object being rendererd. 
            // E.g. model matrix
            uniform_buffer pbr_instance
        }

        // Another resource list. Note of the keyword instance behind list.
        // It indicates that the shader shouldn't load the resources in this
        // list automatically. Its the responsibility of the person that 
        // invokes that shader to bind that resource list. The system will 
        // throw an error if this list is not bound when trying to draw.
        // The mesh_vertices_buffer contains the actual vertex data of the 
        // mesh that gets drawn. Its basically a vertex buffer. Instead of
        // making use of vertex buffers I use a technique called vertex
        // pulling to pull the vertices from storage buffers. This has the
        // benefit to simplify the pipeline handling in some cases. E.g.
        // skeletal animation is easier to achieve this way.
        list instance mesh_data_list
        {
            storage_buffer readonly restrict mesh_vertices_buffer
        }
    }

    // Configure the samplers that get used by this shader.
    // Each texture that gets used can specify which sampler it wants to use.
    sampler_states
    {
        state linear_sampler
        {
            Filter MinMagMipLinear
            AddressU Repeat
            AddressV Repeat
        }
    }

    // The render states encapsulate the blend and raserizer state
    // for the shader.
    render_states
    {
        state solid
        {
            Cull Back
            ZTest LEqual
            ZWrite On
        }
    }

    // The glsl block contains actual shader code. The shader code is just
    // regular GLSL code. The possibility to include other GLSL files was
    // added.
    // Note how it's not necessary to specify any uniform buffers or samplers.
    // This will all be automatically generated from the layout and properties
    // above.
    glsl pbr_gbuffer_vert
    {
        #include "vertex.glsl"

        out vec3 v_ws_pos;
        out vec3 v_ws_normal;
        out vec2 v_uv;

        void main()
        {
            vec3 pos = get_position(gl_VertexID);
            vec3 normal = get_normal(gl_VertexID);
            vec2 uv = get_uv(gl_VertexID);

            vec4 P = u_m_world * vec4(pos, 1.0);
            v_ws_pos = P.xyz / P.w;
            mat3 normal_matrix = transpose(inverse(mat3(u_m_world)));
            v_ws_normal = normalize(normal_matrix * normal);
            v_uv = uv;

            gl_Position = u_m_proj * u_m_view * P;
        }
    }

    // Another glsl block for the fragment shader
    glsl pbr_gbuffer_frag
    {
        #include "common.glsl"
        #include "gbuffer_write.glsl"

        in vec3 v_ws_pos;
        in vec3 v_ws_normal;
        in vec2 v_uv;

        void main()
        {
            vec3 albedo = srgb_to_rgb(texture(u_tx_albedo, v_uv)).rgb *  u_albedo.rgb;
            vec3 emissive = srgb_to_rgb(texture(u_tx_emissive, v_uv)).rgb * u_emissive.rgb;
            vec2 metallic_roughness = texture(u_tx_metallic_roughness, v_uv).yz;
            float roughness = metallic_roughness.x * u_roughness.x;
            float metallic = metallic_roughness.y * u_metallic.x;
            float occlusion = texture(u_tx_ao, v_uv).r;
            vec3 normal = normalize(normal_from_tx());

            write_gbuffer(v_ws_pos, albedo, emissive, normal, metallic, roughness, occlusion, SHADING_MODEL_LIT);
        }
    }

    // The pass is where everything comes together. A shader may have
    // multiple passes.
    // The pass specifies the shaders, resources and render states.
    pass pbr_gbuffer_solid
    {
        resources = pbr_gbuffer_list, mesh_data_list
        vertex_shader = pbr_gbuffer_vert
        fragment_shader = pbr_gbuffer_frag
        rasterizer = solid
    }
}
```

The uniform and storage buffers used in the shader above get configured in text files as well. Here is how the definiton of the globals buffer look like

```txt
uniform_buffer pbr_globals
{
    mat4 u_m_proj
    mat4 u_m_view
    vec4 u_ws_cam_pos
    vec4 u_ws_cam_up
    vec4 u_ws_cam_right
    vec4 u_near_far_resx_resy
    vec4 u_fog_density
    vec4 u_fog_color
    float u_delta_time
    float u_time
    uint u_enable_fog
}
```

From that the engine will create a uniform buffer of the right size and enables the user to set variables by name if they wish. The shader is able to grab that uniform buffer automatically. This is achieved through a centralizes render database. This render database contains all the resources that shaders and renderer may use.

This is how a material looks like that makes use of the shader above:

```txt
material asteroid1_rock1
{
    shader "shaders/gbuffer.sfx"

    properties
    {
        vec4
        {
            name u_albedo
            value (1.000000, 1.000000 , 1.000000, 1.000000)
        }
        vec4
        {
            name u_emissive
            value (0.000000, 0.000000 , 0.000000, 1.000000)
        }
        vec4
        {
            name u_metallic
            value (1.000000, 0.000000 , 0.000000, 0.000000)
        }
        vec4
        {
            name u_roughness
            value (1.000000, 0.000000 , 0.000000, 0.000000)
        }
        texture_2d
        {
            name u_tx_albedo
            value "/textures/asteroid1_rock1_albedo.png"
            sampler linear_sampler
        }
        texture_2d
        {
            name u_tx_normal
            value "/textures/asteroid1_rock1_normal.png"
            sampler linear_sampler
        }
        texture_2d
        {
            name u_tx_metallic_roughness
            value "/textures/asteroid1_rock1_ao-rock1_roughness_metallic.png"
            sampler linear_sampler
        }
        texture_2d
        {
            name u_tx_emissive
            value "__default_tex_emissive__"
            sampler linear_sampler
        }
        texture_2d
        {
            name u_tx_ao
            value "/textures/asteroid1_rock1_ao-rock1_roughness_metallic.png"
            sampler linear_sampler
        }
    }
}
```

Updating uniforms and binding a material looks like this

```c
// Somewhere on startup acquire a reference to the global uniform buffer
ubuffer_globals = dc_renderer_find_ubuffer(renderer, "pbr_globals");

// Before rendering update the buffer
dc_ubuffer_begin_draw(ubuffer_globals, frame_allocator);
dc_ubuffer_set_mat4(ubuffer_globals, "u_m_proj", &proj_mat);
dc_ubuffer_set_mat4(ubuffer_globals, "u_m_view", &view_mat);
// Submit the updated uniforms to the GPU
// This will queue a update buffer command into the command buffer
dc_ubuffer_update(ubuffer_globals, cmd_buffer);
```

Not only the material system is fully configurable through text files. The renderer itself is configurable through text files as well. The whole renderer gets organized over a render graph. This render graph can figure out which pass needs to be executed when and respects the dependencies between passes. 

A simple configuration for the render graph for a forward renderer can be seen below. The render graph consists of pre depth pass and one color pass. Note how the outputs of the pre depth pass are required inputs to the color pass.

```txt
render_graph forward
{
    passes
    {
        pass depth_pre
        {
            outputs
            {
                attachment depth
                {
                    format D32_FLOAT
                    operation clear
                }
            }
        }

        pass opaque
        {
            inputs
            {
                texture depth
            }

            outputs
            {
                attachment final
                {
                    format R8G8B8A8_UNORM
                    operation clear
                }
            }
        }
    }
}

```

All the required textures get automatically created by the render graph. This saves a lot of work when implementing new render stages. And allows to insert barries automatically. The render stages it self get created in C++ and can be registered with the render graph.

A render stage may look like this. Its actually one of the few places where inheritance and virtual functions could be useful. In my engine the render stage is a simple struct that contains function pointers with the option to store some context in it through the user_data pointer.

```c
struct dc_render_stage
{
    void* user_data;

    void (*pre_render) (dc_cmd_buffer*         cmd_buffer,
                        const dc_render_scene* render_scene,
                        void*                  user_data);

    void (*render)     (dc_cmd_buffer*         cmd_buffer,
                        const dc_render_scene* render_scene,
                        u32                    width,
                        u32                    height,
                        f32                    delta_time,
                        void*                  user_data);

    void (*resize)     (dc_gpu_device* gpu_device, 
                        u32            width, 
                        u32            height, 
                        void*          user_data);
};
```

Render stages registration is straight forward:

```c
dc_render_graph_register_stage(render_graph, "gbuffer", gbuffer_stage);
```

The Gbuffer stages render function may looks like this: 

```c
void gbuffer_pass_render (dc_cmd_buffer*         cmd_buffer,
                          const dc_render_scene* render_scene,
                          u32                    width,
                          u32                    height,
                          f32                    delta_time,
                          void*                  user_data)
{
    for (u32 i = 0; i < render_scene.models_count; ++i)
    {
        dc_model_gpu* model        = render_scene.models[i];
        mat4*         model_matrix = &render_scene.model_matrices[i];

        for (u32 i = 0; i < model->meshes_count; ++i)
        {
            dc_mesh_gpu* mesh     = &model->meshes[i];
            dc_material* material = mesh.material;

            mat4 normal_mat = mat4_transpose(mat4_inverse(world));

            // Update instance uniform buffer
            dc_ubuffer_begin_draw(instance_ub, render_scene->frame_allocator);
            dc_ubuffer_set_mat4(instance_ub, "u_m_world", &world);
            dc_ubuffer_set_mat4(instance_ub, "u_m_normal", &normal_mat);
            dc_ubuffer_update_buffer(instance_ub, cmd_buffer);

            // Update the resource list that gets attached to the
            // shader for this draw call.
            // The resource list contains the information where the shader
            // can find the vertex buffer for this mesh
            dc_material_set_resource_list(material,
                                          "mesh_data_list",
                                          mesh.vertices_list);

            // Bind the material for the draw call
            // This makes sure that all resource lists have been bound
            // Note how the name matches the name of the pass specified
            // in the shader file.
            dc_shader* shader = material->shader;
            i32 pass_index = dc_shader_pass_index(shader, "pbr_gbuffer_solid") 
            dc_material_bind(material,
                             pass_index,
                             cmd_buffer,
                             render_scene->frame_allocator);

            // Bind the index buffer and execute the draw call
            dc_cmd_buffer_bind_index_buffer(cmd_buffer, mesh.index_buffer);
            dc_cmd_buffer_draw_indexed(cmd_buffer,
                                       DC_TOPOLOGY_TRIANGLE, 
                                       mesh.index_count, 
                                       0,
                                       0,
                                       0,
                                       0);
        }
    }
}
```

This system works quite well and allows me to create new shaders and render stages quite easily without much work. There are still some rough edges and it will need to proof itself when I come to implement more complicated rendering algorithms.

For the shader and material system and render graph I was initially inspired by [these blog posts](https://jorenjoestar.github.io/post/data_driven_rendering_pipeline/) and [this book](https://www.packtpub.com/en-us/product/mastering-graphics-programming-with-vulkan-9781803244792?srsltid=AfmBOorl5bd9_kCEJXe-iW4Qm0qGJ3hZZECcM7qG6Niyl2nRYHpyBYHU).

## Game UI

For the Game UI I created a very simple system that is powerful enough to display good looking UIs. One of the main challenges with UI systems is the font rendering. I support in my engine ttf and bitmap fonts. For ttf file parsing I use the [STB single header library](https://github.com/nothings/stb/blob/master/stb_truetype.h). The usage of this library is not straightforward in my oppinion, but [these videos](https://www.youtube.com/watch?v=zvGIp-S2mxA) got me started well.

For the layout system I was inspired by [this blog post](https://edw.is/learning-vulkan/#ui).

## Asset Management

My asset management is very simple. The games I plan to make will be very simple and don't require clever asset management. If they ever will, I will need to come up with something more sophisticated.

I have a central asset store where different asset loaders can be registered with and if the user wants to load a specific asset during runtime the user queries the asset store for a asset and the asset store will pick the matching loader depending on the file extension. The asset store will only invoke the loader if the asset hasn't been loaded yet. When the asset gets loaded for the first time it will be stored in a simple hash map to avoid redundant loading calls.

For most of my assets I use a custom binary file format. This allows me to load the assets on startup fast and I can get rid of some runtime dependencies. To create this custom assets I have a separate asset importer tool that can take in a source file and will then produce a file that the engine can understand.

### ECS

It may come as a surprise, but I don't use an ECS (Entity Component System) or other scene system to organize my game entities. My current games are too small and don't require such sophisticated systems. Currently I create simple struct for game entites and this has been working for me well so far.

### Deployment

A important aspect of making a game is to actual ship it. Games have the additional challenge in comparison with regular applications that the bundle a lot additinal data with the executable. For example textures, meshes, shaders etc. There are different approaches to this problem.

One possibility is to zip the game executable together with all its asset files and hand the user over that zip file. They then can extract that zip file and launch your game. The downside of that approach is that it is quite involved for your users to try out the game. They need to understand that they have to unzip the file and then they need to ensure that they keep the data files together with your executable because otherwise the executable will likley not find the data which will probably lead to unexpected behavior and crashes.

A better approach is to bundle all the assets into a zip file for before shipping and then hand the user your executable and the zip data file. The user just has to execute executable, but they still need to make sure the zip data file and the executable are in the same directory to allow the executable find the data files.

The installation process can be simplified with a installer that puts the files in the right directories. However, this is quite an involved process.

A third option, and the option that I did choose is to just bundle all the game assets into the executable itself. This way only a single executable needs to be handed over to the user. A downside of this is that the executable gets quite large. That approach may only work for smaller games. But it is very easy for users to just download a executable and execute it. There is no much room for potential errors.

My packinging process works as follows:

- A Python scripts zips and compresses all the game data and puts it into a zip file
- A C script will then convert that zip file into valid C code. It will be just a byte array filled with the data from the zip file.
- Then the script compiles the game together with that generated C file.
- The game code of course has some logic in it to detect that the game data is integrated into the executable.
- The game code will just mount the byte array into the virtual file system. The virtual file system uncompresses the zip file and makes it available as a filesystem.

## Final Words

This completes a brief overview over my engine.
