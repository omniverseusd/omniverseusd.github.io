# Schemas

Schemas are a fundamental part in USD: a schema defines which properties (with fallback/default values) and which metadata a prim should have. A prim that uses the `Cube` schema is expected to have a `size` attribute and USD will also generated methods of the syntax `Get[attr_name]Attr()/Create[attr_name]Attr`, i.e. `GetSizeAttr()/CreateSizeAttr()`.

```admonish example
The [`UsdPhysics.usda` schema](https://github.com/PixarAnimationStudios/OpenUSD/blob/release/pxr/usd/usdPhysics/schema.usda) in the official pixar USD repo defines a `collisionEnabled` attribute in the `PhysicsCollisionAPI` class so that each prim which inherits from the `PhysicsCollisionAPI` is going to have that attribute (which semantically controls whether the prim will participate or not in the physics engine simulation of collisions)
```

There are two types of schemas:

* `Typed Schemas` (also called `IsA-schemas`): these can impart a typeName to a `UsdPrim`. An example is `Cube`: in the text `usda` the defined prim as type `cube`

    ```python
    def Cube "Cube" # type is 'Cube'
    {
        float3[] extent = [(-50, -50, -50), (50, 50, 50)]
        double size = 100
    }
    ```

    The type of a prim can be inspected with `IsA` (which in C++ is a templated function for the inspecting type)

    ```python
    cube : UsdGeom.Cube = UsdGeom.Cube.Define(root_stage, "/World/Cube")
    extent = [(-50, -50, -50), (50, 50, 50)]
    cube.GetExtentAttr().Set(extent)
    cube.GetSizeAttr().Set(100)
    ..

    cube_prim : Usd.Prim = cube.GetPrim()

    print(cube_prim.IsA(UsdGeom.Cube)) # True
    ```

    By using the `Tf` module (recall that it's used for type-related operations) we can do type manipulations as well and inspect the typed schema further (although this stuff is more advanced):

    ```python
    # Query the typename and look it up in the internal USD registry of all known schemas up to
    # this point
    prim_type_name = cube_prim.GetTypeName()
    print(prim_type_name) # "Cube", this is the textual name of the prim type name
    # Get the Tf.Type (Tf is for internal type operations) from this 'Cube' type name
    prim_tftype_type : Tf.Type = Usd.SchemaRegistry.GetTypeFromName(prim_type_name)
    # Use it to define a new variabile (get its python class with `pythonClass`)
    # which points to the same Cube prim instance
    myObj : prim_tftype_type.pythonClass = cube
    print(myObj.GetSizeAttr().Get()) # "100"

    # Get the textual representation of its class type receiving 'cube_prim' in its constructor
    prim_typed_schema = prim_tftype_type.pythonClass(cube_prim)
    print(prim_typed_schema) # "UsdGeom.Cube(Usd.Prim(</World/Cube>))"
    ```
    Note that this kind of schemas can be `Concrete` or `Abstract/non-concrete` which means that it can be instantiated directly (e.g. `Cube`) or cannot be instantiated directly and you'll have to either define or find a subclass which adds the missing required pieces (e.g. `UsdGeom.Imageable`).

* `API Schemas`: these do **not** define a prim type and **do not** contribute to a prim's definition but rather add methods and properties/metadata to have the prim behave in a certain way. There are two types of API schemas:
    * `non-applied` schemas (we'll take a look at the `kind` non-applied schema later) which only provide an API to set and get data for a prim (and you usually just use that API to access it, e.g. the `kind` non-applied schema which is basically a "is-this-the-topmost-parent-prim-of-a-large-and-complete-3d-model?" field)
    * `single-apply` schemas: adds properties to a prim's definition, e.g. the `UsdCollisionAPI` for physics collision behavior. These can be queried via `UsdPrim::HasAPI<..>()` or the equivalent Python `Usd.Prim.HasAPI("CollisionAPI")` method.
    * `multiple-apply` schemas: these can be applied to a prim more than once requiring an "instance name" to distinguish them, a typical example is the `UsdShadeMaterialBindingAPI` multiple-apply schema which can be applied multiple times to a prim to bind different materials to different subsets of a geometry.


    Here are some examples of what we just learned:

    ```python
    root_stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")

    # Add a cube to the scene

    xform : UsdGeom.Xform = UsdGeom.Xform.Define(root_stage, Sdf.Path("/World"))
    cube : UsdGeom.Cube = UsdGeom.Cube.Define(root_stage, "/World/Cube")
    extent = [(-50, -50, -50), (50, 50, 50)]
    cube.GetExtentAttr().Set(extent)
    cube.GetSizeAttr().Set(100)
    UsdPhysics.CollisionAPI.Apply(cube.GetPrim())
    environment_xform = UsdGeom.Xform.Define(root_stage, "/World/Environment")
    dome_light = UsdLux.DomeLight.Define(root_stage, "/World/Environment/DomeLight")
    dome_light.CreateIntensityAttr(1000)

    cube_prim : Usd.Prim = cube.GetPrim()

    # Get the typed schema of the cube prim
    prim_type_name = cube_prim.GetTypeName()
    prim_tftype_type : Tf.Type = Usd.SchemaRegistry.GetTypeFromName(prim_type_name)
    prim_typed_schema = prim_tftype_type.pythonClass
    # Here the type can be used, as we saw before, to define references or new cube variables

    ## API Schemas ##

    # Get the non-applied schema 'kind'
    non_applied_api_schema = Usd.ModelAPI(cube_prim)
    # use it
    non_applied_api_schema.SetKind(Kind.Tokens.subcomponent) # we'll see what this means later

    # Get the applied schema CollisionAPI
    applied_api_schema = UsdPhysics.CollisionAPI(cube_prim)
    # use it
    applied_api_schema.GetCollisionEnabledAttr().Set(True)
    ```

    E.g. of `usda` for an API single-appy schema `PhysicsRigidBodyAPI`:

    ```python
    #usda 1.0
    ()
    over "Kitchen_set"
    {
        over "FlowerPotA_5" (
            prepend apiSchemas = ["PhysicsRigidBodyAPI"]
        )
        {}
    }
    ```


And here is a table which intuitively maps the concepts that we've just learned about schemas to OOP (object oriented programming) concepts you might be familiar with:

| USD Concept      |   Similar OOP Concept |
|------------------|-----------------------|
| Non-concrete typed schema | Abstract class (non-instantiable directly) |
| Concrete typed schema | (Instantiable) class |
| Non-applied API Schema| Provide methods to access/set non-defining properties (and you cannot manipulate its state directly) but **does not contribute to type** in any way |
| Single applied API Schema | A member variable inside your class that you can use - it has its state and methods |
| Multi-applied API Schema | An array of member variables inside your class that you can use - each element inside the array has its own state even though they all have the same methods |

Some of these can also be queried from the `Usd.SchemaRegistry()` - a registry that contains the list of all schema names, types and fallback/default values for all known schemas

```python
registry = Usd.SchemaRegistry()
print(registry.IsTyped(UsdGeom.Cube))  # True
print(registry.IsTyped(UsdGeom.Imageable)) # True
print(registry.IsAbstract(UsdGeom.Imageable))  # True
print(registry.IsAbstract(UsdGeom.Cube)) # False
print(registry.IsConcrete(UsdGeom.Imageable))  # False
print(registry.IsConcrete(UsdGeom.Cube)) # True
print(registry.IsTyped("UsdGeomImageable"))  # True
print(registry.IsTyped("UsdGeomCube"))  # True
print(registry.IsAppliedAPISchema("CollisionAPI"))  # True
print(registry.IsMultipleApplyAPISchema("CollectionAPI"))  # True
print(registry.GetSchemaKind("Cube"))  # pxr.Usd.SchemaKind.ConcreteTyped
print(registry.GetSchemaKind("Imageable"))  # pxr.Usd.SchemaKind.AbstractTyped
```

Lastly, let's take a look at a more complex example that removes a typed API schema from a [primspec](../chapter4/primspecs_and_attributes.md) and uses [List Composition](./chapter4/list_composition.md):

```python
root_stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")

def removeAPI(prim, api_name):
    if prim.IsInstanceProxy() or prim.IsInPrototype():
        return # invalid prim

    # Get a primspec on the root stage
    editTarget = root_stage.GetEditTarget()
    primSpec = editTarget.GetPrimSpecForScenePath(prim.GetPath())

    listOp = primSpec.GetInfo(Usd.Tokens.apiSchemas)

    # Look for the API in the prepended/appended/explicit lists
    if api_name not in listOp.prependedItems:
        if api_name not in listOp.explicitItems:
            if api_name not in listOp.appendedItems:
                return # not found, we're good

    # Create a new list with whatever it was already present MINUS the api_name we want to remove
    newPrepended = listOp.prependedItems
    newPrepended.remove(api_name)
    listOp.prependedItems = newPrepended

    result = listOp.ApplyOperations([])
    newListOp = Sdf.TokenListOp()
    newListOp.prependedItems = result # Reassignment is needed here due to legacy reasons
    # Write back the primspec again
    primSpec.SetInfo(Usd.Tokens.apiSchemas, newListOp)

# Add a cube to the scene
xform : UsdGeom.Xform = UsdGeom.Xform.Define(root_stage, Sdf.Path("/World"))
cube : UsdGeom.Cube = UsdGeom.Cube.Define(root_stage, "/World/Cube")
cube_prim : Usd.Prim = cube.GetPrim()
extent = [(-50, -50, -50), (50, 50, 50)]
cube.GetExtentAttr().Set(extent)
cube.GetSizeAttr().Set(100)
# Apply a CollisionAPI
UsdPhysics.CollisionAPI.Apply(cube_prim)
environment_xform = UsdGeom.Xform.Define(root_stage, "/World/Environment")
dome_light = UsdLux.DomeLight.Define(root_stage, "/World/Environment/DomeLight")
dome_light.CreateIntensityAttr(1000)

# Remove the CollisionAPI
removeAPI(cube_prim, "PhysicsCollisionAPI")
print(cube_prim.GetMetadata("apiSchemas")) # SdfTokenListOp(Explicit Items: [])
```

## Custom schemas

As previously stated USD is _extensible_. This means that custom schemas can be defined.
This is rather common when dealing with a custom pipeline from a DCC (digital content creation software) that involves USD: a developer defines his own schemas to create custom prim types/API so that the prims have sets of attributes relevant to the DCC in question or to the kind of workflow intended.

There is a [pretty good tutorial on the OpenUSD official website](https://openusd.org/release/tut_generating_new_schema.html#schema-generation-prerequisites) regarding schema generations, but we'll summarize the steps here for clarity:

* One would usually first figure out what kind of schema he's after (is it a multiple-apply schema? a single-apply one?). Then a `usda` schema would usually be defined, e.g. the [`UsdPhysics schema`](https://github.com/PixarAnimationStudios/OpenUSD/blob/release/pxr/usd/usdPhysics/schema.usda). This is called the _schema definition file_.

* A schema definition file can be contained within a `USD plugin`, indicating that schema definitions and associated code (if not codeless) will be included in the resulting C++ and Python libraries. A `USD plugin` is a shared library object (e.g. .dll or .so) that USD applications can load via the `Plugin registry`

    ![](../images/chapter4/plugin_listing.png)

* A script called `usdGenSchema` provided by the official pxr repo can be used to generate C++ classes (and/or python bindings)

    ```shell
    $ usdGenSchema schema.usda .

    Processing schema classes:
    SimplePrim, ComplexPrim, ParamsAPI
    Loading Templates
    Writing Schema Tokens:
            unchanged extras/usd/examples/usdSchemaExamples/tokens.h
            unchanged extras/usd/examples/usdSchemaExamples/tokens.cpp
            unchanged extras/usd/examples/usdSchemaExamples/wrapTokens.cpp
    Generating Classes:
            unchanged extras/usd/examples/usdSchemaExamples/simple.h
            unchanged extras/usd/examples/usdSchemaExamples/simple.cpp
            unchanged extras/usd/examples/usdSchemaExamples/wrapSimple.cpp
            unchanged extras/usd/examples/usdSchemaExamples/complex.h
            unchanged extras/usd/examples/usdSchemaExamples/complex.cpp
            unchanged extras/usd/examples/usdSchemaExamples/wrapComplex.cpp
            unchanged extras/usd/examples/usdSchemaExamples/paramsAPI.h
            unchanged extras/usd/examples/usdSchemaExamples/paramsAPI.cpp
            unchanged extras/usd/examples/usdSchemaExamples/wrapParamsAPI.cpp
            unchanged extras/usd/examples/usdSchemaExamples/plugInfo.json
    Generating Schematics:
            unchanged extras/usd/examples/usdSchemaExamples/generatedSchema.usda
    ```
* Stuff is then compiled to build the plugin shared library object that can be loaded

    ```shell
    $ cmake --build . --target install --config Release
    ```

* Finally the environment variable `PXR_PLUGINPATH_NAME` can be used to indicate the location of the plugin's `resources` directory so it can be loaded from conforming USD-based applications (e.g. Kit for Omniverse apps).

