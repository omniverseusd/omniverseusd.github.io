# Diving into References, VariantSets, Payloads, Inherits and Specializes

In this section we'll explore practical code examples for the other composition arcs that allow USD to compose scene prims.

## References

In the same fashion as before, let's create a prim with a `references` metadata that references another layer in a different USD file and open it in OV Composer (this script can be run in the `Script Editor` in OV Composer directly)

```python
from pxr import Sdf, UsdGeom, Usd, UsdLux, Gf
import omni.usd
import carb

BASE_DIRECTORY = "/tmp"  # This is where the .usda files will be saved

root_stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")
cube_stage : Usd.Stage = Usd.Stage.CreateInMemory("CubeLayer.usda")

# Create a Cube "Cube" prim in the cube stage and a light
xform : UsdGeom.Xform = UsdGeom.Xform.Define(cube_stage, Sdf.Path("/World"))
cube : UsdGeom.Cube = UsdGeom.Cube.Define(cube_stage, "/World/Cube")
extent = [(-50, -50, -50), (50, 50, 50)]
cube.GetExtentAttr().Set(extent)
cube.GetSizeAttr().Set(100)
# This time put the light under the "/World" prim - by referencing "/World" we will also
# import the lights as well since they're under "/World"
environment_xform = UsdGeom.Xform.Define(cube_stage, "/World/Environment")
dome_light = UsdLux.DomeLight.Define(cube_stage, "/World/Environment/DomeLight")
dome_light.CreateIntensityAttr(1000)
# Export to file
cube_stage.GetRootLayer().Export(BASE_DIRECTORY + "/CubeLayer.usda")

# A pseudo-root prim (usually the '/' prim) exists solely to namespace all prims' paths under
# this prefix (e.g. "/World"). A UsdStage always has a pseudo-root prim (unless there was an
# error opening it)
root_prim : Usd.Prim = root_stage.GetPseudoRoot()
# Verbose to showcase API usage for "/RefPrim". Creates a generic UsdPrim (it will default
# to being an Xform)
ref_prim : Usd.Prim = root_stage.DefinePrim(str(root_prim.GetPath()) + "RefPrim")
# This would NOT work because as soon as this script finishes executing, the temporary stages
# would be destroyed (they're weakrefs). Let's open and reference the usd file for those
# layers instead
# nope: ref_prim.GetReferences().AddReference(cube_stage.GetRootLayer().identifier)
loaded_layer = Sdf.Layer.FindOrOpen(BASE_DIRECTORY + "/CubeLayer.usda")
ref_prim.GetReferences().AddReference(
    loaded_layer.identifier, # which in this case it's just the relative file path string
    "/World") # The prim which needs to be mapped at the ref_prim also needs to be specified

# Export root stage to file
root_stage.GetRootLayer().Export(BASE_DIRECTORY + "/RootLayer.usda")

# Issue an 'open-stage' command to avoid doing this manually and free whatever stage
# was previously owned by this context
omni.usd.get_context().open_stage(BASE_DIRECTORY + "/RootLayer.usda")
```

Here's the result in OV Composer: a prim is added to the root prim that references the outside layer (OV Composer adds a small orange arrow to the icon of the xform to indicate that the prim has a `reference` metadata and that references something during the stage composition)

![](../images/chapter3/reference_prim_in_ov_composer.png)

If we inspect the generated USD files we can see the `reference` metadata

```python
$ cat /tmp/CubeLayer.usda
#usda 1.0

def Xform "World"
{
    def Cube "Cube"
    {
        float3[] extent = [(-50, -50, -50), (50, 50, 50)]
        double size = 100
    }

    def Xform "Environment"
    {
        def DomeLight "DomeLight"
        {
            float inputs:intensity = 1000
        }
    }
}

$ cat /tmp/RootLayer.usda
#usda 1.0

def "RefPrim" (
    prepend references = @/tmp/CubeLayer.usda@</World>   # Here it is
)
{
}

```

Note what happens if we save the USD file with OV Composer as _flattened_:

```python
$ cat /tmp/Flattened.usda
#usda 1.0
(
    doc = """Generated from Composed Stage of root layer /tmp/RootLayer.usda
"""
)

def Xform "RefPrim"
{
    def Cube "Cube"
    {
        float3[] extent = [(-50, -50, -50), (50, 50, 50)]
        double size = 100
    }

    def Xform "Environment"
    {
        def DomeLight "DomeLight"
        {
            float inputs:intensity = 1000
        }
    }
}

```

i.e. only the _composed_ elements are retained and the composition arcs are resolved into a single, flattened, hierarchy. This is very useful when dealing with e.g. huge Nucleus USD files which reference multiple-networks-scattered USD files and you want to save a copy (albeit probably _huge_) of the USD scene for debugging purposes on your local workstation: you flatten it out and save it to disk.


Note that overriding attributes works exactly as before: even if the `cube` had an opinion in the original `CubeLayer`

```python
# Give it a red color (quick way to apply a 'debug' color without shaders or materials - accesses primvar variables)
cube.GetPrim().CreateAttribute("primvars:displayColor", Sdf.ValueTypeNames.Color3fArray).Set([(1.0, 0.0, 0.0)])
# Export to file
cube_stage.GetRootLayer().Export(BASE_DIRECTORY + "/CubeLayer.usda")
```

we could have overridden that attribute with an override (that would appear as a _delta on the referenced layer_) in the root layer

```python
# Give it a blue color override in the root layer
override_cube_prim : Usd.Prim = root_stage.OverridePrim("/RefPrim/Cube")
override_cube_attr = override_cube_prim.CreateAttribute("primvars:displayColor", Sdf.ValueTypeNames.Color3fArray)
override_cube_attr.Set([(0.0, 0.0, 1.0)])
# Export root stage to file
root_stage.GetRootLayer().Export(BASE_DIRECTORY + "/RootLayer.usda")
```

```admonish info
In the code above we apply a `displayColor` in the original `cube` stage by accessing that attribute via the `UsdGeomPrimvar` schema. This is a schema that allows to access and modify attributes which are _specific_ to geometry prims (i.e. prims like cameras and such do not implement such schema), stuff like `visibility` or `interpolation` are under control of this schema. It's a quick way to visualize a simple shaded color for a geometric prim without setting a material.
```

The final color of the cube would have been blue: root layer wins in this case. Had the blue opinion come from a variant (lower in the `LIVRPS` ordering), the reference opinion would have won.


## VariantSet

Now let's see an example of `VariantSet` attribute override in action together with a `reference` attribute override and let's remember the `LIVRPS` order of opinions evaluation:

```python
from pxr import Sdf, UsdGeom, Usd, UsdLux, Gf
import omni.usd
import carb

BASE_DIRECTORY = "/tmp"  # This is where the .usda files will be saved

root_stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")
cube_stage : Usd.Stage = Usd.Stage.CreateInMemory("CubeLayer.usda")

# Create a Cube "Cube" prim in the cube stage and a light

xform : UsdGeom.Xform = UsdGeom.Xform.Define(cube_stage, Sdf.Path("/World"))
cube : UsdGeom.Cube = UsdGeom.Cube.Define(cube_stage, "/World/Cube")
extent = [(-50, -50, -50), (50, 50, 50)]
cube.GetExtentAttr().Set(extent)
cube.GetSizeAttr().Set(100)
environment_xform = UsdGeom.Xform.Define(cube_stage, "/World/Environment")
dome_light = UsdLux.DomeLight.Define(cube_stage, "/World/Environment/DomeLight")
dome_light.CreateIntensityAttr(1000)
# In the cube layer (the one where the cube is defined and that will be referenced by RootLayer),
# the cube has originally a red color
cube.GetPrim().CreateAttribute("primvars:displayColor", Sdf.ValueTypeNames.Color3fArray).Set([(1.0, 0.0, 0.0)])
# Export to file
cube_stage.GetRootLayer().Export(BASE_DIRECTORY + "/CubeLayer.usda")

# Now as before set up a "/ReferencedCube" prim which references the "/World/Cube" in the CubeLayer

ref_prim : Usd.Prim = root_stage.DefinePrim("/ReferencedCube")
loaded_layer = Sdf.Layer.FindOrOpen(BASE_DIRECTORY + "/CubeLayer.usda")
ref_prim.GetReferences().AddReference(
    loaded_layer.identifier, # which in this case it's just the relative file path string
    "/World/Cube") # The prim which needs to be mapped at the ref_prim also needs to be specified

# At this point if graph composition were to end, the cube would just have an opinion in the referenced layer
# and would be a red colored one.

# This is where things get interesting: set up a variant set with sub-variants in the root layer
# (a VariantSet is like a new combo box to switch something in the prim, e.g. "colors for the car" or
# "level of damage of the car" or "types of wheel rims")
variant_set = ref_prim.GetVariantSets().AddVariantSet("differentColorsVariantSet")
# A variant is owned by a variant set, think of these as the different items that you can choose from the
# combo box (which is the VariantSet)
variant_set.AddVariant("blueColor")
variant_set.AddVariant("greenColor")

# Set the active variant
variant_set.SetVariantSelection("blueColor")
with variant_set.GetVariantEditContext():  # Set up an edit context (this is just like changing the authoring layer)
    # Give the cube in the root layer a blue color
    override_cube_prim : Usd.Prim = root_stage.OverridePrim("/ReferencedCube")
    override_cube_attr = override_cube_prim.CreateAttribute("primvars:displayColor", Sdf.ValueTypeNames.Color3fArray)
    override_cube_attr.Set([(0.0, 0.0, 1.0)])

variant_set.SetVariantSelection("greenColor")
with variant_set.GetVariantEditContext():
    # Give the cube in the root layer a green color
    override_cube_prim : Usd.Prim = root_stage.OverridePrim("/ReferencedCube")
    override_cube_attr = override_cube_prim.CreateAttribute("primvars:displayColor", Sdf.ValueTypeNames.Color3fArray)
    override_cube_attr.Set([(0.0, 1.0, 0.0)])

# Select the active variant after editing them
variant_set.SetVariantSelection("blueColor")

# Export root stage to file
root_stage.GetRootLayer().Export(BASE_DIRECTORY + "/RootLayer.usda")

# Issue an 'open-stage' command to avoid doing this manually and free whatever stage
# was previously owned by this context
omni.usd.get_context().open_stage(BASE_DIRECTORY + "/RootLayer.usda")
```

If you execute this code, you should get a reference prim but with a _blue_ color: even though the red color is authored and there's an opinion in the referenced layer, the `Variant` opinion is stronger in the `LIVRPS` ordering and therefore wins (note that OV Composer has a very handy combo box in the `Properties` pane that shows the `VariantSet` along with its current active `Variant`)

![](../images/chapter3/variantset_and_references.png)

USDA code is also provided for completeness

```python
$ cat /tmp/RootLayer.usda
#usda 1.0

def "ReferencedCube" (
    prepend references = @/tmp/CubeLayer.usda@</World/Cube>
    variants = { # Active variants for multiple VariantSets are stored in the 'variants' metadata
        string differentColorsVariantSet = "blueColor"
    }
    prepend variantSets = "differentColorsVariantSet" # the additional VariantSets
)
{
    variantSet "differentColorsVariantSet" = { # Definition for the VariantSet which will override displayColor
        "blueColor" {
            color3f[] primvars:displayColor = [(0, 0, 1)]

        }
        "greenColor" {
            color3f[] primvars:displayColor = [(0, 1, 0)]

        }
    }
}

$ cat /tmp/CubeLayer.usda
#usda 1.0

def Xform "World"
{
    def Cube "Cube"
    {
        float3[] extent = [(-50, -50, -50), (50, 50, 50)]
        color3f[] primvars:displayColor = [(1, 0, 0)] # This opinion will be weaker due to LIVRPS composition
        double size = 100
    }

    def Xform "Environment"
    {
        def DomeLight "DomeLight"
        {
            float inputs:intensity = 1000
        }
    }
}

```

