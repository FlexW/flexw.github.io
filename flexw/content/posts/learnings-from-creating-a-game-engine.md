+++
title = 'Learnings From Creating a Game Engine'
date = 2024-07-28T10:00:00+02:00
draft = true
+++

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

My renderer is a Clustered Deferred Renderer built on top of OpenGL but in a way that would allow to use different graphic APIs as well. Why did I use OpenGL? First I'm very familiar with the API can work with it quickly. I think OpenGL is high level enough to not care about some specifics that are not interessting to me (yet) and secondly it provide enough low level access to control the things I care about. Overall, I think it's a great API.

A big problem with OpenGL is that its a big state machine. Meaning if you change some state in it every subsequent calls will be affected by the changes as well. This makes it especially difficult to reason about it in bigger projects. Another downside is that OpenGL functions can only be invoked from a single thread. To work around it created a rendering API on top of it that works almost stateless and in a way mimics Vulkans API.

The rendering API allows to create rendering resources like buffers, textures, resource lists, pipelines and command buffers.

During a frame all rendering commands get recorded in one or multiple command buffers (its possible to record them in parallel). This command buffer then gets submitted to the rendering backend at the end of a frame. The rendering backend then processes the commands and translates them into OpenGL commands and executes them. This approach allows to perform optimizations on the order of draw calls and in addition to parallelize the submission of draw calls and it allows running the OpenGL backend in a separate thread. Currently I do none of these optimizations because there is simply no need for it.

In addition, for this engine I decided to experiment with a material system that is fully configurable with text files. For this I wrote my own lexer and parsers and ended up with a system that lets me configure shaders, materials, their state and the needed resources for rendering in simple text files. To complement the configurable shader and material system I have also the possibility to configure uniform buffers and storage buffers in a text file. The engine will create these buffers automatically on startup.

For the general architecture of the renderer I decided to go with a Clustered Deferred Renderer. This decision was driven by mainly two thoughts: I wanted to be able to render many lights and I wanted to have a proofen architecture. Clustered Deferred Rendering is what most engines do these days so I do the same. For this part I also experimented with the idea to configure the complete render pipeline in a text file as well. I ended up with render graph that can be configured through a text file. That makes it very easy to add new render stages. The render graph will handle the texture creation and will execute the render stage at the right time for myself which is very convinient.

For the shader and material system and render graph I was mainly inspired by [these blog posts](https://jorenjoestar.github.io/post/data_driven_rendering_pipeline/) and [this book](https://www.packtpub.com/en-us/product/mastering-graphics-programming-with-vulkan-9781803244792?srsltid=AfmBOorl5bd9_kCEJXe-iW4Qm0qGJ3hZZECcM7qG6Niyl2nRYHpyBYHU).

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
