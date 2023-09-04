---
title: "What is Usd"
weight: -20
resources:
  - name: ov_muting_layers_example
    src: "ov_muting_layers_example.gif"
    title: Example of muting USD layers in Omniverse
---

USD stands for "Universal Scene Description" and at this point is an umbrella term that includes an open standard (https://openusd.org/) to collaboratively edit and create 3D scenes and assets, apply physically correct properties to the 3D assets, describe the materials to apply, define the hierarchies (e.g. which asset goes under which other asset to create the 3D model of a car), define how properties vary with time (e.g. for an animation of a 3D model) and much more.

Among other things the USD standard also describes a file format (.usd files) which defines how to save that information to a single or multiple files that can be shared with other users locally or across a network to collaborate on authoring (i.e. 'editing') and visualizing 3D scenes.

A consortium called the [Alliance for OpenUSD](https://aousd.org/) has recently been created and among other tasks they're working on creating a formal specification (much like the C++'s standard) for OpenUSD.

## What is the goal of USD

There used to be no consensus among modeling softwares (e.g. Blender, 3D Studio Max or Maya) and programs operating on assets needed to compose 3D scenes (e.g. programs to aid in material designing like Adobe Substance Painter) on how to communicate with each other (especially if from different vendors). These programs are commonly referred to as _DCC tools_, i.e. "Digital Content Creator" tools.

USD aims at filling that gap and providing a single source of truth to exchange files seamlessly and let multiple people work on the same 3D scene/model at the same time.

If a 3D content creator program like Blender or 3D Studio Max or Maya has a USD _connector_ available (be it a plugin, an addon or whatever integrated or external piece of code usually developed by NVIDIA), it means the program is able to export its content (be it a 3D model, a texture, or even just plain text data) to a USD file. And that file can be later read and imported by another program which has a USD connector. Plus there are tools to visualize those USD files, to edit them, to compose them and so on.

## USD is complex but comprehensive

USD is a very descriptive, complex and powerful 3D representation format, some of its key points:

* It was initially developed by Pixar and later open-sourced (free to use under Apache 2.0), so it’s rather battle-tested for **large-scale** usage

* It is often addressed as the 'Photoshop of 3D graphics' since it uses **layers** to compose a final component of a scene (e.g. the color of an asset can be overridden in a stronger layer, but the other colors are retained as well - one can always go back to it or compose them together) and it allows for multiple people editing the same 3D scene collaboratively and/or in real-time (if the underlying software allows it). E.g. see this image where layer2 (a USD file itself) which gives the cube a red color with a stronger opinion is muted and the renderer transitions the cube to a blue color (given by layer1 which has a weaker opinion on the color given by the layers' hierarchy)

<!-- spellchecker-disable -->

{{< img name="ov_muting_layers_example" lazy=false size=origin >}}

<!-- spellchecker-enable -->

* It is **extensible**. USD allows for new _schemas_ to be defined: a schema is a textual document of rules. E.g. there could be a 'light schema' which defines light properties like the light’s color, intensity and direction. The schema should afterwards be parsed and interpreted by a software (like Omniverse) which can later load a USD file with primitives (for now just think of ‘primitives’ as 3D assets) which have that schema applied to them (much like fulfilling an interface contract in programming languages or inheriting from a pure virtual class) and have the properties defined by that schema defined and instantiated into them. So to make it simple a software can read a ‘LightBulb’ prim (3D asset) which has the 'Light' schema applied on it, and therefore it must have `color`, `intensity` and `direction` defined in order for the renderer to actually render it on the scene. This is the [real light schema](https://github.com/PixarAnimationStudios/OpenUSD/blob/release/pxr/usd/usdLux/schema.usda) in usda format (USD format in ASCII - i.e. human readable, not binary nor compressed), as you can see it is pretty complex but there is no need to grasp that complexity right now.

    Schemas are a very powerful mechanism which allows the USD format to expand, evolve and represent complex 3D objects properties (e.g. **physics properties**!).

Pixar has also open-sourced with the same license a reference implementation (i.e. nothing stops you from conforming to the specifications and writing a USD implementation yourself) of the USD standard, i.e. they provide a rather large C++ library hosted at [github.com/PixarAnimationStudios/USD](https://github.com/PixarAnimationStudios/OpenUSD) that users can download, compile, contribute to, etc.. the library is written in C++ but python bindings are also available in that same repository to call USD APIs from Python programs.

It has to be noted that this library doesn’t provide any renderer in itself, i.e. even if you clone the repository, build it, generate the python bindings and write code to generate a cube on an empty stage and save it to a USD file as follows

```python
from pxr import Usd, UsdGeom

# Create a new stage
stage = Usd.Stage.CreateNew('helloWorld.usd')

# Define a new Cube primitive
cube = UsdGeom.Cube.Define(stage, '/HelloWorldCube')

# Set the size of the cube
cube.GetExtentAttr().Set([(1.0, 1.0, 1.0)])

# Save the stage
stage.GetRootLayer().Save()
```

you will be able to execute this python USD code and get a USD output file with a stage with a cube on it, but you will not have any way to visually inspect and render that cube. USD defines another rather complex specification for a rendering architecture called Hydra that renderer programmers can abide by to have their own renderer integrate with USD scenes. There is however a small tool called [usdView](https://docs.omniverse.nvidia.com/usdview/latest/overview.html) in the same official USD repo based on PyQt that allows you to quickly render USD stages (mostly for debugging purposes and to understand how USD works).

In the next pages we will introduce a powerful tool that we'll be using to render the contents of USD files and to execute USD python commands dynamically on a stage.