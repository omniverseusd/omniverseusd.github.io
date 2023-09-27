# Debugging

Debugging USD can be challenging, there are some `Tf` debugging facilities that can be used to make this job easier and print detailed log in composition phases and inspect what's going on within multiple plugins

Here's an excerpt from [USD-Cookbook](https://github.com/ColinKennedy/USD-Cookbook/blob/33eac067a0a62578934105b19a2b9d8e4ea0646c/features/enable_debugging/python/debugging.py)

```python
from pxr import Sdf, UsdGeom, Usd, UsdLux, Gf, Tf
import omni.usd
import carb

stage : Usd.Stage = Usd.Stage.CreateInMemory("RootLayer.usda")

# Redirect debug output to stdout, it can be redirected to a file as well
Tf.Debug.SetOutputFile(sys.__stdout__)

# Actual symbols are defined in C++ across many files.
# You can query them using `Tf.Debug.GetDebugSymbolNames()` or by
# searching for files that call the `TF_DEBUG_CODES` macro in C++.
# (Usually this is in files named "debugCodes.h").
symbols = Tf.Debug.GetDebugSymbolNames()

# Check if debug symbols are enabled
# (on my machine, they're all False by default)
for symbol in symbols:
    print(symbol)
    print(Tf.Debug.IsDebugSymbolNameEnabled(symbol))

# A more detailed full description of everything
print("Descriptions start")
print(Tf.Debug.GetDebugSymbolDescriptions())
print("Descriptions end")

# Enable change processing so we can see something happening
# You can also use glob matching. Like "USD_*" to enable many flags
# at once.
Tf.Debug.SetDebugSymbolsByName("USD_CHANGES", True)

stage.DefinePrim("/SomePrim")  # This line will print multiple messages to stdout
```

Note that per-plugin debugging should be enabled if you're doing this in OV Composer. To enable _all_ of the debug codes that start with the prefix `PLUG_` the following can be used

```shell
$ TF_DEBUG="PLUG_*" /home/alex/.local/share/ov/pkg/create-2023.2.0/omni.create.sh
...
# script above gets executed..
HandleLayersDidChange received (stage with rootLayer @anon:0x12035890:RootLayer.usda@, sessionLayer @anon:0x1c4ada40:RootLayer-session.usda@)
</SomePrim> in @anon:0x12035890:RootLayer.usda@ changed.
Changed field: specifier
Adding paths that use </SomePrim> in layer @anon:0x12035890:RootLayer.usda@: [ /SomePrim ]

ProcessPendingChanges (stage with rootLayer @anon:0x12035890:RootLayer.usda@, sessionLayer @anon:0x1c4ada40:RootLayer-session.usda@)
Did Change Significantly: /SomePrim
Recomposing: /SomePrim
```

A comprehensive list of debug codes can be found here: [Debugging USD](https://lucascheller.github.io/VFX-UsdSurvivalGuide/core/profiling/debug.html).