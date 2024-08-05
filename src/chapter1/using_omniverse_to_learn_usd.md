# Using Omniverse to learn USD

Dabbling with USD is the quickest and easiest way to learn it, much like copy-pasting and changing your first C++ hello world source code. One of the best ways to do this is to use [NVIDIA Omniverse](https://www.nvidia.com/en-us/omniverse/) for this.
It is highly recommended that you download and install Omniverse Standard (specifically Omniverse USD Composer) [for free from the NVIDIA website](https://www.nvidia.com/en-us/omniverse/download/).

Let's spend some words introducing what is Omniverse exactly. Feel free to skip this page if you know that already.

## What is Omniverse
Omniverse is a platform and a series of technologies developed by NVIDIA around the USD standard (although they’re rapidly evolving in other directions as well).

Omniverse comprises technologies and applications to work with 3D graphics, collaborating on creating 3D assets and scenes, using AI to create stunning visual effects or improve the process of creating 3D contents, adding real-time and physically correct physics behaviors to 3D contents, rendering in a physically-correct way with ray tracing or path tracing in real-time, etc.

NVIDIA doesn’t impose any workflow or dictate how Omniverse tools and technologies should be used (they can be used to create photorealistic render images that you later use commercially, they can be used to let multiple 3D artists work on a 3D scene simultaneously without interfering with each other’s modifications, they can be used to ‘predict’ the mechanical ‘wear’ in a ‘digital twin’ 3D representation of a mechanical part in 3D with accurate physics after many simulation steps, they can be used to create a [server-side web service](https://docs.omniverse.nvidia.com/dev-guide/latest/programmer_ref/services.html) which renders something complex and streams the result as a video back to the user’s browser, etc.).

The foundation of many Omniverse applications is called Kit and it’s an extensible framework and SDK developed by NVIDIA which provides a series of libraries and APIs to let users write extensions (i.e. kit-based libraries written in Python, C++, both or in other languages as well) so that they can use NVIDIA’s best-in-breed technologies (e.g. RTX raytracers, PhysX, AI integrations, etc.) to do useful graphical work for them. [Official docs for Kit can be found here](https://docs.omniverse.nvidia.com/dev-guide/latest/kit-architecture.html).

An example: a user can write a Kit extension ([here's a good list of extensions used in Omniverse](https://docs.omniverse.nvidia.com/extensions/latest/index.html)) which acts as a web service: whenever a HTTP request is received, the extension can fire up a complex 3D scene, generate the assets or the scene requested [via the HTTP request with chatGPT](https://www.youtube.com/watch?v=mFazJsjUUSo), render it and send the rendered result back to the user. Another extension could show up a small UI panel in the Omniverse Composer application (more on this later) which allows a 3D artist to click a button, process the scene geometries with an AI application and apply effects, shaders and much more. Extensions are the core of Omniverse as we will later discover.

## Composer & Presenter
Two of the most famous reference applications (and some of the very first ones a newcomer might try out) in Omniverse are USD Composer (formerly Create) and USD Presenter (formerly View).

The former is a 3D authoring program which allows users to compose complex scenes from 3D assets, applying physical properties to them, simulating and rendering, applying photorealistic materials and much more

[![](../images/chapter1/create_joints_authoring_video.png)](https://www.youtube.com/watch?v=3QjFjpUooXI)

Composer/Create is usually not equipped with 3D creation tools to model single 3D assets (think Blender) but rather orchestrates composition of a USD scene from external assets (although it could even become a modeling tool with the right extensions).

Presenter/View instead focuses on visualizing already composed environments and inspecting USD scenes (it doesn’t feature advanced authoring tools as Composer).

There is some overlap in the UI elements of the two applications, as there is in some of the core extensions used for a simple reason: both Composer and Presenter are just a bunch of highly complex Kit based extensions. Kit is both a SDK that programmers can leverage to build their own extensions and also a **portable cross-platform executable** that can ‘bootstrap’ their extensions with a bare minimum skeleton environment. Composer and Presenter are both instances of the Kit platform executable but they load different extensions. These extensions can also be altered (and dependencies broken, if the user so desires..), changed or custom ones loaded. This is the beauty of Omniverse: you can compose it in any way you want. You’re not satisfied with a particular workflow? Create your own extension and fire it up from a Kit CLI command or from the Composer UI panel.

## Nucleus

Omniverse isn't just a collection of Kit extensions though but much more. For instance: to foster collaboration between lots of 3D artists working together on a complex 3D scene (think Pixar..), together with the USD specification and file format (which features "photoshop layer-like behavior" for 3D contents) Omniverse also provides Nucleus which is a distributed object storage optimized for graphical assets.

With nucleus users can reference `omniverse://asset_paths` from external repositories or internal organization repos, use or modify those assets and seamlessly have them streamed back to other users which might be using them. Dynamically.

## Pricing and requirements

Two things newcomers usually care a lot about: pricing and requirements.

Please check [this page](https://learnomniverse.github.io/chapter1/what_is_omniverse.html#pricing-and-requirements) for a detailed explanation of pricing and requirements for Omniverse.


## Support, learning, official resources

Official channels to learn more about Omniverse, post questions regarding its official applications and main extensions (e.g. related to omni.physx) and get in touch with the great NVIDIA Omniverse community (friendly and available, NVIDIA is doing its best to foster a good community) are the [Omniverse discord server](https://forums.developer.nvidia.com/t/omniverse-discord-server-is-live/178422), the [YouTube Omniverse channel](https://www.youtube.com/c/nvidiaomniverse), the [developer blog articles](https://developer.nvidia.com/blog/tag/omniverse/) and, for critical bugs/issues, the [official Omniverse forum](https://forums.developer.nvidia.com/c/omniverse/) (less chatty, more support-y).

Any non-Omniverse related question should not be asked in the above channels but rather in the [NVIDIA customer support forum](https://www.nvidia.com/en-us/support/).
