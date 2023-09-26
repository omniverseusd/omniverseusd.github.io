# List Composition

In USD there is a concept called [List Editing](https://openusd.org/release/glossary.html#list-editing) which can be seen in action when defining metadata (e.g. `apiSchemas` for a prim):

```python
$ cat ./MyUSDLayer.usda
#usda 1.0
()
over "Kitchen_set"
{
    over "FlowerPotA_5" (
        prepend apiSchemas = ["PhysicsRigidBodyAPI"] # List-Editing in action
    )
    {}
}
```

We've already seen the `prepend` keyword but let's take a deeper look at its meaning.
A list can make use of `append` and/or `prepend` and/or `delete` and/or `explicit` actions when defining its items to change the **order** of the items (remember that in composition, just like in the layers stack, the order is **important**: in the layer stack, the higher the layer, the stronger the opinions - the root layer usually wins in fact). The same applies to these lists in USD: `["/cube", "/sphere", "/cone"]` here in this list `/cube` has a **stronger** opinion than `/cone`. If you keep order in mind, [the official documentation explanation](https://openusd.org/release/glossary.html#list-editing) makes sense:

> * **append** another value or values to the back of the resolved list; if the values already appear in the resolved list, they will be reshuffled to the back. An appended composition arc in a stronger layer of a LayerStack will therefore be weaker than all of the arcs of the same type appended from weaker layers, by default; however, the Usd API’s for adding composition arcs give you some flexibility here.
>
> * **delete** a value or values from the resolved list. A “delete” statement can be speculative, that is, it is not an error to attempt to delete a value that is not present in the resolved list.
>
> * **prepend** another value or values on the front of the resolved list; if the values already appear in the resolved list, they will be shuffled to the front. A prepended composition arc in a weaker layer of a LayerStack will still be stronger than any arcs of the same type that are appended from stronger layers.
>
> * **reset to explicit** , which is an “unqualified” operation, as in references = @myFile.usd@. This causes the resolved list to be reset to the provided value or values, ignoring all list ops from weaker layers.

To summarize in simpler words: if you have a stronger layer (think the root layer at the top of the layer stack) and it defines some list property in a `append` field, it means that the items _must_ go to the end of the list and therefore the root layer wants that value to be **weaker** in the hierarchy (not deleted, but weaker than any other value of the same type).
If the bottom layer (the weakest) adds an item to a list in the `prepend` field it means that weakest layer wants that item to be as strong as possible. But since it comes from the weakest layer, only if no other layer has any other opinion on it that is going to happen. If a higher layer (which is stronger) re-defines it in the same `prepend`, the game is already lost for the weakest layer: its item will _not_ be at the forefront of the list because another layer added something else (unless it added the same exact value). `delete` does exactly what you think: it _deletes_ an item from the list, and this is still subject to whatever precedence the layer has when applying it on a list. `explicit` causes the list to be reset to whatever values are explicitly provided in there - but still: this is only as good as the precedence of the currently applying layer.

Here's a coding example:

```python
from pxr import Sdf, UsdGeom, Usd, UsdLux, Gf, UsdPhysics
import omni.usd
import carb

BASE_DIRECTORY = "/tmp"  # This is where the .usda files will be saved

# Create a temporary stage in memory for the root layer and another sublayer
top_layer : Usd.Stage = Usd.Stage.CreateInMemory("TopLayer.usda")
middle_layer : Usd.Stage = Usd.Stage.CreateInMemory("MiddleLayer.usda")
bottom_layer : Usd.Stage = Usd.Stage.CreateInMemory("BottomLayer.usda")

sphere : UsdGeom.Sphere = UsdGeom.Sphere.Define(top_layer, Sdf.Path("/World/Sphere"))
sphere_prim = sphere.GetPrim()
middle_override_prim = middle_layer.OverridePrim("/World/Sphere")
bottom_override_prim = bottom_layer.OverridePrim("/World/Sphere")

# Get a edit target for the top layer, get a prim spec for the /World/Sphere and
# *low-level* and *manually* add some items to the apiSchemas metadata
editTarget = top_layer.GetEditTarget()
primSpec = editTarget.GetPrimSpecForScenePath("/World/Sphere")
newListOp = Sdf.TokenListOp()
newListOp.deletedItems = ["PhysicsRigidBodyAPI"]
primSpec.SetInfo(Usd.Tokens.apiSchemas, newListOp)

# The above is conceptually equivalent to this (higher level) but only to _add_ an API:
#
# edit_target = top_layer.GetEditTargetForLocalLayer(top_layer.GetRootLayer())
# top_layer.SetEditTarget(edit_target)
# UsdPhysics.RigidBodyAPI.Apply(sphere_prim)
#
# but we get to modify deletedItems directly with the code above.

# Now add some prependedItems to the middle layer
editTarget = middle_layer.GetEditTarget()
primSpec = editTarget.GetPrimSpecForScenePath("/World/Sphere")
newListOp = Sdf.TokenListOp()
newListOp.prependedItems = ["PhysicsCollisionAPI", "PhysicsMassAPI"]
primSpec.SetInfo(Usd.Tokens.apiSchemas, newListOp)

# And finally add some appendedItems and a deletedItem to the bottom layer
editTarget = bottom_layer.GetEditTarget()
primSpec = editTarget.GetPrimSpecForScenePath("/World/Sphere")
newListOp = Sdf.TokenListOp()
newListOp.prependedItems = ["PhysicsRigidBodyAPI"]
newListOp.appendedItems = ["PhysicsMassAPI", "PhysicsArticulationRootAPI"]
primSpec.SetInfo(Usd.Tokens.apiSchemas, newListOp)

# Export all layers to file and set the sublayers relationships

top_layer.GetRootLayer().subLayerPaths.append(BASE_DIRECTORY + "/MiddleLayer.usda")
top_layer.GetRootLayer().subLayerPaths.append(BASE_DIRECTORY + "/BottomLayer.usda")


# Now query the COMPOSED final stage, not just the primspecs
# output:
# SdfTokenListOp(Explicit Items: [PhysicsCollisionAPI, PhysicsMassAPI, PhysicsArticulationAPI])
print(sphere_prim.GetMetadata("apiSchemas"))
# it basically went on like this:
#
# top_layer:    "I'm the most important, make sure that PhysicsRigidBodyAPI is DELETED"
# middle_layer: "I'm halfway important, it's imperative for me that PhysicsCollisionAPI and PhysicsMassAPI
#                are added as soon as possible to the list"
# bottom_layer: "I'm not important.. anyway if I have any voice in the matter I'd like PhysicsRigidBodyAPI
#                to be present and at the front of the list. Oh and also PhysicsMassAPI and
#                PhysicsArticulationRootAPI should be present but at the bottom of the list (both not very
#                important in my opinion).
#
# and the USD engine resolves things according to each layer's importance.


middle_layer.GetRootLayer().Export(BASE_DIRECTORY + "/MiddleLayer.usda")
bottom_layer.GetRootLayer().Export(BASE_DIRECTORY + "/BottomLayer.usda")
top_layer.GetRootLayer().Export(BASE_DIRECTORY + "/TopLayer.usda")

omni.usd.get_context().open_stage(BASE_DIRECTORY + "/TopLayer.usda")
```

The above is equivalent to (taken from [here](https://lucascheller.github.io/VFX-UsdSurvivalGuide/core/composition/listeditableops.html)):
```python
from pxr import Sdf
### Merging basics ###
path_list_op_layer_top = Sdf.PathListOp.Create(deletedItems = [Sdf.Path("/cube")])
path_list_op_layer_middle = Sdf.PathListOp.Create(prependedItems = [Sdf.Path("/disc"), Sdf.Path("/cone")])
path_list_op_layer_bottom = Sdf.PathListOp.Create(prependedItems = [Sdf.Path("/cube")], appendedItems = [Sdf.Path("/cone"),Sdf.Path("/sphere")])

result = Sdf.PathListOp()
result = result.ApplyOperations(path_list_op_layer_top)
result = result.ApplyOperations(path_list_op_layer_middle)
result = result.ApplyOperations(path_list_op_layer_bottom)
# Notice how on merge it makes sure that each sublist does not have the values of the other sublists, just like a Python set()
print(result) # Returns: SdfPathListOp(Deleted Items: [/cube], Prepended Items: [/disc, /cone], Appended Items: [/sphere])
# Get the flattened result. This does not apply the deleteItems, only ApplyOperations does that.
print(result.GetAddedOrExplicitItems()) # Returns: [Sdf.Path('/disc'), Sdf.Path('/cone'), Sdf.Path('/sphere')]
```

List composition can sometimes be daunting to newcomers but if explained properly it should be quite straightforward to grasp.
