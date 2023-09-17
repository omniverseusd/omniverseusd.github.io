# Foundational classes

You might have noticed from the few python code listings in the previous sections that the USD API is somewhat scattered throughout different classes/submodules with different prefixes from the pxr module:

```python
from pxr import Usd, Sdf, UsdGeom, Vt, Gf

# Some examples of modules into the pxr
Sdf.Layer.FindOrOpen("/tmp/usd_file.usda")
Usd.Stage.Open(..)
xformable = UsdGeom.Xformable(prim)
xformable.AddTranslateOp().Set(Gf.Vec3d(0.0, 180.0, 0.0))
positions = Vt.Vec3fArray(positions_list)
material = UsdShade.Material(material_prim)
```

Each prefix represents a different module with a specific purpose and functionality. Here is a brief overview of the main ones (there are many others though):

| Module | Meaning of acronym |Description |
|--------|---------|------------|
|[Sdf](https://openusd.org/release/api/sdf_page_front.html)     | Scene Description Format | Low-level core scene description format and data model API for USD. It provides classes for representing layers, paths, schemas, attributes, references, variants, and more. Sdf is the foundation of USD and is used by all other modules. These routines come in handy when you want to reason on specific parts (opinions) coming from layers that end up composing a prim on the stage (and on how they interact and are combined together). |
|[Gf](https://openusd.org/release/api/gf_page_front.html)      | Graphics Foundation      | Basic geometric types and operations, such as vectors, matrices, transforms, colors, quaternions, etc. Gf is used by other modules to perform computations and manipulations on geometric data. |
|[Tf](https://openusd.org/release/api/tf_page_front.html)      | Tools Foundation         | Low-level utilities and foundation classes, such as python module initialization, string manipulation, error handling, debugging, type traits, smart pointers, etc. mostly OS-independent. Tf is used by other modules to implement common functionality and patterns. |
|[Vt](https://openusd.org/release/api/vt_page_front.html)      | Value Types              | Classes useful to abstract away types and have collections of values, such as arrays, dictionaries, etc. Vt is used by other modules to store and access data in a generic way.
| [Usd](https://openusd.org/release/api/usd_page_front.html) | Universal Scene Description (Core) | Core Usd module: high-level scenegraph API that exposes USD's composition features to application code. It provides classes for accessing and editing stages, prims, properties, etc. Usd is the main entry point for most USD applications. |
| [UsdGeom](https://openusd.org/release/api/usd_geom_page_front.html) |  USD Geometry Schema | Core geometry schemas for USD (we'll revisit schemas later): it provides classes for representing common geometric primitives, such as meshes, curves, points, cameras, etc. UsdGeom also defines concepts such as transformation spaces, visibility, purpose, etc. |
| [UsdShade](https://openusd.org/release/api/usd_shade_page_front.html)| USD Shading Schema | Shading schema for USD: it provides classes for representing materials, shaders, textures, etc. UsdShade also defines a network of connectable nodes for describing shading effects. |
| [UsdLux](https://openusd.org/release/api/usd_lux_page_front.html)| USD Lighting Schema | Lighting schema for USD: it provides classes for representing lights, light filters, light links, etc. UsdLux also defines a set of built-in light types and filters that can be used by renderers. |


Many other modules in the USD C++ API provide additional functionality and domain-specific schemas, [you can take a look at them here](https://openusd.org/release/api/index.html).

Knowing at least the module which owns a method or a class can help understanding what areas that API is operating on.

## C++ and Python in USD

As we've already stated the USD API has a reference implementation in C++ with [a quite comprehensive API documentation](https://openusd.org/release/api/index.html). USD was born C++-first and Python bindings were added later. This means sometimes there are small differences between the C++ and Python APIs:

```cpp
// Get the current stage (use weakrefs, the context owns a strong reference to the stage)
pxr::UsdStageWeakPtr stage = omni::usd::UsdContext::getContext()->getStage();
if (!stage) {
    return 1;
}

// Traverse the stage and print out all paths for the found prims
pxr::UsdPrimRange range = stage->Traverse();
for (pxr::UsdPrimRange::iterator iter = range.begin(); iter != range.end(); ++iter) {
    // Get the current prim
    pxr::UsdPrim prim = *iter;
    if (!prim)
        continue;

    pxr::SdfPath path = prim.GetPath();
    CARB_LOG_INFO("Found a prim with path: %s", path.GetText()); // Carbon API: Omniverse-specific
}
```

```python
# Get the current stage (weakref)
stage = omni.usd.get_context().get_stage()
if not stage:
    return False

# Traverse the stage and print out all paths for the found prims
for prim in self.stage.Traverse():
    # Get the current prim
    if not prim:
        continue

    path = prim.GetPath()
    carb.log_info(f"Found a prim with path {str(path)}")  # Carbon API: Omniverse-specific
```

If you don't remember the reason why we're always getting a weakref to the stage, read up on the previous [context and stages section](./context_stage_and_layers.md).

There is a chapter dedicated to [Translating Between the OpenUSD C++ and Python APIs](https://docs.omniverse.nvidia.com/workflows/latest/openusd-developer/api-comparison.html) in the official Omniverse documentation which might provide more insights when working with porting C++ and/or Python USD code.
Omniverse supports both native C++ extensions and Python extensions (together with extensions having both Python and C++ in them, a common pattern is for example writing UI code in Python and core processing logic in C++ for performance reasons).
In this book however we'll mainly focus on Python APIs to dive into learning OpenUSD. All the concepts are easily applicabile to the C++ API as well.
