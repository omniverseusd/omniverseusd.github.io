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
Two of the most famous applications (and some of the very first ones a newcomer might try out) in Omniverse are USD Composer (formerly Create) and USD Presenter (formerly View).

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

**Omniverse is free to use for individuals, but a license must be purchased for team use**: [Omniverse Licensing](https://www.nvidia.com/en-us/omniverse/download/).

More in detail (from the official discord):

```admonish quote
Omniverse users are welcome to sell their extensions for whatever they please. The end users must have a license of Omniverse to use their extensions with, but that can be the free Omniverse Individual version.
```

So programmers are free to write and sell their Omniverse extensions. Users can buy and use those extensions as long as they do it abiding by the Omniverse license (i.e. if they're working as a team of 20 people with Omniverse, an enterprise license must be purchased). If they're working as individuals (or teams of 2 people), no license is necessary and Omniverse is totally free.

What about 3D content I create with e.g. Omniverse Composer? Can I sell a rendered video of a Physical simulation made with Omniverse?

```admonish quote
Content and or code\extensions\apps created using OVI (Omniverse Individual license, i.e. abiding by the 2-users-tops requirement) for small teams, using desktops or cloud resources is allowed and can be used for commercial purposes.
```

So yes: you can create a video using Omniverse and you can sell it for whatever you want.

Can I use Omniverse in my own private cloud?

```admonish quote
For the free version, you are allowed to put Omniverse in the cloud for your own purposes. For example, you are allowed to put OV apps on Azure or AWS VM, create 3D projects and render out those projects using Omniverse Farm which can also be on an Azure VM for free.

The EULA is designed that once you scale the number of users working together and you need support, you should get the enterprise license.

Other licensing example, You can also use the Omniverse Individual version to create, build, sell your own extensions and or apps for free. The user leveraging that extension or app just needs to follow the same EULA.
```

The only other restriction pertains to letting users use your "abiding OVI individual license" Omniverse apps as cloud services:

```admonish quote
Lets say for example, you put USD Composer in the cloud and allow anyone to use it for free as a streamed application. This would not be allowed, using OVI as a service to users outside your company.
```

For any other question or clarification please read the final paragraph of this post and get in contact with NVIDIA sales for a special license tailored to your needs: [omniverse-license-questions@nvidia.com](mailto:omniverse-license-questions@nvidia.com) will get you in touch with a developer relations manager that can work with you.

Regarding Omniverse requirements, each application that works on the Omniverse platform might have different system requirements. The suggested way to get up-to-date information is to browse the NVIDIA website for the app you’re specifically interested in, e.g. [the USD Composer/Create page](https://www.nvidia.com/en-us/omniverse/apps/create/) lists requirements like a RTX class card as minimum viable hardware.


## Support, learning, official resources

Official channels to learn more about Omniverse, post questions regarding its official applications and main extensions (e.g. related to omni.physx) and get in touch with the great NVIDIA Omniverse community (friendly and available, NVIDIA is doing its best to foster a good community) are the [Omniverse discord server](https://forums.developer.nvidia.com/t/omniverse-discord-server-is-live/178422), the [YouTube Omniverse channel](https://www.youtube.com/c/nvidiaomniverse), the [developer blog articles](https://developer.nvidia.com/blog/tag/omniverse/) and, for critical bugs/issues, the [official Omniverse forum](https://forums.developer.nvidia.com/c/omniverse/) (less chatty, more support-y).

Any non-Omniverse related question should not be asked in the above channels but rather in the [NVIDIA customer support forum](https://www.nvidia.com/en-us/support/).
