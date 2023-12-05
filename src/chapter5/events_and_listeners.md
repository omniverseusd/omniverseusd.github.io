# Events and Listeners

USD features an event system to notify and take action in case of added prims, removed prims, attribute changes, a stage was just loaded, etc.

```admonish warn
Your callback functions registered as listeners should be **fast**: think of ISVs (interrupt service vectors) - their actions should either be fast or another thread/entity should be delegate to take heavy actions later. This is by design since they run in the thread where the listener was created and that's often the main thread.
```

The basic structure of a USD event listener is as follows (most of the facilities are in the `Tf` module - see [Tools Foundations](../chapter2/foundational_classes_and_py_cpp_differences.md)):

```python
def ObjectsChanged_callback(notice, sender):
    print(notice.GetResyncedPaths())
    print(notice.GetChangedInfoOnlyPaths())

listener = Tf.Notice.RegisterGlobally(Usd.Notice.ObjectsChanged, ObjectsChanged_callback)

# do some USD manipulations here..

listener.Revoke() # cleanup
```

* ResyncedPaths are **structural changes** that can invalidate entire subtrees of UsdObjects (both prims and properties). These happen when you delete or add a prim, or even if you add a property or metadata. Anything that relates to _composition_ is considered a structural change.

* ChangedInfoOnly are **non-structural** changes, e.g. when you change an attribute's or metadata's value.

Here is a code example of registering a listener and spewing out a long list of information regarding what just happened. At a high level overview, the following is going to happen:

1. A listener is registered
2. A `/World` Xform is added to the scene, this will cause a resync (it's a _structural_ change)
3. `/World/Cube` is added, structural
4. Some attributes (as mandated by the `Cube` typed schema) are changed in `/World/Cube`, this is non-structural
5. `/World/Environment` and `/World/Environment/DomeLight` are created.. same as before..
6. The attribute `/World/Cube.myCustomFloatAttribute` is created. Structural change.
7. The default value of attribute `/World/Cube.myCustomFloatAttribute` is changed. Non-structural change.
8. The listener is revoked

Here's the full code and output with explicative comments on what's happening

```python
from pxr import Sdf, UsdGeom, Usd, UsdLux, Gf, Tf
import omni.usd
import carb

BASE_DIRECTORY = "/tmp"  # This is where the .usda files will be saved

root_stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")

def ObjectsChanged_callback(notice, sender):
    stage = notice.GetStage()
    print("---")
    print(">", notice, sender)
    print(">> (notice.GetResyncedPaths) - Updated paths", notice.GetResyncedPaths())
    print(">> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes", notice.GetChangedInfoOnlyPaths())

    prim = stage.GetPrimAtPath("/World/Cube")
    if prim:
        # path #1

        # Check if a specific UsdObject was affected
        print(">> (notice.AffectedObject) - Something changed for", prim.GetPath(), notice.AffectedObject(prim))
        print(">> (notice.ResyncedObject) - Updated path for", prim.GetPath(), notice.ResyncedObject(prim))
        print(">> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly", prim.GetPath(), notice.ChangedInfoOnly(prim))
        print(">> (notice.HasChangedFields) - Attribute/Metadata HasChanges", prim.GetPath(), notice.HasChangedFields(prim))
        print(">> (notice.GetChangedFields) - Attribute/Metadata ChangedFields", prim.GetPath(), notice.GetChangedFields(prim))

    attr = stage.GetAttributeAtPath("/World/Cube.myCustomFloatAttribute")
    if attr:
        # path #2

        # Check if a specific UsdObject was affected
        print(">> (notice.AffectedObject) - Something changed for", attr.GetPath(), notice.AffectedObject(attr))
        print(">> (notice.ResyncedObject) - Updated path for", attr.GetPath(), notice.ResyncedObject(attr))
        print(">> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly", attr.GetPath(), notice.ChangedInfoOnly(attr))
        print(">> (notice.HasChangedFields) - Attribute/Metadata HasChanges", attr.GetPath(), notice.HasChangedFields(attr))
        print(">> (notice.GetChangedFields) - Attribute/Metadata ChangedFields", attr.GetPath(), notice.GetChangedFields(attr))

listener = Tf.Notice.RegisterGlobally(Usd.Notice.ObjectsChanged, ObjectsChanged_callback)

# Add some prims to the scene
xform : UsdGeom.Xform = UsdGeom.Xform.Define(root_stage, Sdf.Path("/World"))
cube : UsdGeom.Cube = UsdGeom.Cube.Define(root_stage, "/World/Cube")
extent = [(-50, -50, -50), (50, 50, 50)]
cube.GetExtentAttr().Set(extent)
cube.GetSizeAttr().Set(100)
environment_xform = UsdGeom.Xform.Define(root_stage, "/World/Environment")
dome_light = UsdLux.DomeLight.Define(root_stage, "/World/Environment/DomeLight")
dome_light.CreateIntensityAttr(1000)

# Create an attribute
cube_prim : Usd.Prim = cube.GetPrim()
attr = cube_prim.CreateAttribute("myCustomFloatAttribute", Sdf.ValueTypeNames.Float)
attr.Set(42.8)

listener.Revoke()

# Export root stage to file
root_stage.GetRootLayer().Export(BASE_DIRECTORY + "/RootLayer.usda")

# Issue an 'open-stage' command to avoid doing this manually and free whatever stage
# was previously owned by this context
omni.usd.get_context().open_stage(BASE_DIRECTORY + "/RootLayer.usda")
```

Output:

```python
# listener is registered here

# /World Xform is created
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []

# /World/Cube is created, path #1 begins to always be executed from now on
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Cube')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube True
>> (notice.ResyncedObject) - Updated path for /World/Cube True
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube True
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube ['specifier', 'typeName']

# /World/Cube.extent attribute is created (this is defined in the typed schema)
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Cube.extent')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/Cube.extent attribute has its value changed
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths []
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes [Sdf.Path('/World/Cube.extent')]
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/Cube.size is created
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Cube.size')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/Cube.size has its value changed
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths []
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes [Sdf.Path('/World/Cube.size')]
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/Environment is created
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Environment')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/DomeLight is created... we'll skip all this part since it's similar to what we just explained
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Environment/DomeLight')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Environment/DomeLight.inputs:intensity')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths []
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes [Sdf.Path('/World/Environment/DomeLight.inputs:intensity')]
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []

# /World/Cube.myCustomFloatAttribute is created - path #2 is also executed from now on
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths [Sdf.Path('/World/Cube.myCustomFloatAttribute')]
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes []
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []
>> (notice.AffectedObject) - Something changed for /World/Cube.myCustomFloatAttribute True
>> (notice.ResyncedObject) - Updated path for /World/Cube.myCustomFloatAttribute True
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube.myCustomFloatAttribute False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube.myCustomFloatAttribute True
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube.myCustomFloatAttribute ['custom']

# /World/Cube.myCustomFloatAttribute has its default value changed
---
><pxr.Usd.ObjectsChanged object at 0x7feb381c4ea0> Usd.Stage.Open(rootLayer=Sdf.Find('anon:0x1db9dcf0:RootLayer.usda'), sessionLayer=Sdf.Find('anon:0xfb39dd0:RootLayer-session.usda'), pathResolverContext=<invalid repr>)
>> (notice.GetResyncedPaths) - Updated paths []
>> (notice.GetChangedInfoOnlyPaths) - Attribute/Metadata value changes [Sdf.Path('/World/Cube.myCustomFloatAttribute')]
>> (notice.AffectedObject) - Something changed for /World/Cube False
>> (notice.ResyncedObject) - Updated path for /World/Cube False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube False
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube False
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube []
>> (notice.AffectedObject) - Something changed for /World/Cube.myCustomFloatAttribute True
>> (notice.ResyncedObject) - Updated path for /World/Cube.myCustomFloatAttribute False
>> (notice.ChangedInfoOnly) - Attribute/Metadata ChangedInfoOnly /World/Cube.myCustomFloatAttribute True
>> (notice.HasChangedFields) - Attribute/Metadata HasChanges /World/Cube.myCustomFloatAttribute True
>> (notice.GetChangedFields) - Attribute/Metadata ChangedFields /World/Cube.myCustomFloatAttribute ['default']

# Listener is revoked here
```

As you can see the USD event system is quite comprehensive and allows you to inspect carefully what's happening on the scene.

Here's a link to a more complex example involving creating an Omniverse extension printing out prim paths in the viewport directly: this also uses a `Usd.Notice.ObjectsChanged` listener to be notified if anything changes in the selected prim so it can react and update accordingly: [How to make an extension to display Object Info](https://github.com/NVIDIA-Omniverse/workshop-siggraph-2022/blob/main/exts/omni.example.ui_scene.object_info/Tutorial/object_info.tutorial.md#how-to-make-an-extension-to-display-object-info).

One last thing to pay attention to: stage callbacks are usually handled by [USD contexts](../chapter2/context_stage_and_layers.md), therefore the right place to find such callbacks would be taking a look at USD contexts documentations (usually provided by the application you're using USD in), in Omniverse that would be [omni.usd.USDContext](https://docs.omniverse.nvidia.com/kit/docs/omni.usd/latest/omni.usd/omni.usd.UsdContext.html):

```python
stage: Usd.Stage = Usd.Stage.Open(some_usd_url)
cache = UsdUtils.StageCache.Get()
# Retrieve a long int id from the singleton cache for all local USD clients
stage_id = cache.Insert(stage).ToLongInt()
def on_stage_opened(result, err):
    if result is True:
        print(f"There were errors opening the stage: {err}")
    else:
        print("no errors, stage opened!")
omni.usd.get_context().attach_stage_with_callback(stage_id=stage_id, on_finish_fn=stage_opened_fn)
```
