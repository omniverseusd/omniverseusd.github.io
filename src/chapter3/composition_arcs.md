# Composition Arcs

We already saw in the previous chapter how multiple sublayers each one with different opinions are composed together when referenced by another layer to compose a final stage.

There are other ways and operators that can be used to determine how layer stacks and their opinions are combined together, here's an intuitive overview:

* **Sublayers** - we've seen these already, similar to Photoshop layers. Each one can be backed by a USD file on disk and provide - according to their order in the layer stack - different opinions.
* **References** - they allow to put entire layers under prims to maximize prims reuse
* **VariantSet** - switchable states for a prim: think of multiple colors, materials and even different wheel rims for a car in a configurator scene
* **Payload** - _lazily loaded references_: these can be loaded or unloaded at runtime on user request (think LoD - Level of Detail - a high resolution model can be avoided to be loaded entirely if it makes sense on an already-quite-heavy USD scene)
* **Inherits** - similar to inheritance in object oriented programming: prims can inherit from other prims. Changes in the base prim reflect immediately on the prims that derive from it.
* **Specializes** - e.g. a `Metal` material can have some properties specialized to create a `RustyMetalMaterial`. Very similar to `RustyMetalMaterial` inheriting from `Metal`, but specialized properties are **always** the winning ones even if in some layer/other_composition we override those `Metal`'s base properties with a stronger opinion: a bit similar to CSS's `!important` a specialization of a property cannot be overridden. With `inherit` you're able to override a property multiple times instead. That's the main difference.

Even if `Specializes` wins over other overrides, this composition arc is the _last one_ and _weakest_ to be scanned. This is by design.

In the next sections we'll take a deeper coding look at each of these USD composition techniques, but first an overview into their evaluation order.

## LIVRPS

This acronym means `L`ocal(and Sublayers), `I`nherits, `V`ariantSets, `R`eferences, `P`ayloads and `S`pecializes.
This is the order in which the USD engine evaluates opinions.

When evaluating the value of an attribute on a prim or a metadata value, we iterate over `PrimSpecs` (which can be thought as a _partial view of a prim in a layer_ or rather: the part of a prim which resides in a specific layer and, together with all the other `PrimSpec`s for that prim in the other layers, contributes in composing the final prim on the stage with all the 'winning' opinions for all attributes and metadata) in the **LIVRPS** order.

```admonish example
The attribute `radius` for a type `Sphere` sphere prim needs to be evaluated on the final composed stage.
1. First we search for an attribute opinion in the Local/Sublayers stack (same rules as before: root layer wins, otherwise sublayers in the order in which they appear under the root layer, etc.). If we find a winning one, we abort the search and use it.
2. No opinion was found? We continue by scanning the `inherits` and recursively start again from 1 with the new added layers. Specializations are ignored though.
3. No opinion was found? We scan `variants` and recursively start from 1. Specializations are ignored though.
4. No opinion was found? We scan `References` and recursively start from 1. Specializations are ignored though.
5. No opinion was found? We even scan the optional `Payload`s and recursively start from 1. Specializations are ignored though.
6. Still no opinion was found? We scan `Specializes` for an opinion.

If after point 6 still no opinion could be found, we just take the **default value** for that attribute/metadata.
```
