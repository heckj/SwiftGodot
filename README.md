# SwiftGodot

SwiftGodot provides Swift language bindings for the Godot 4.0 game
engine using the new GDExtension system.

SwiftGodot can be used to either build extension that can be added
to an existing Godot project, where your code is providing services
to the game engine, or it can be used as an API with SwiftGodotKit
which embeds Godot as an application that is driven directly from
Swift.

Driving Godot from Swift has the advantage that on MacOS you can
debug your code from Xcode as well as the Godot code.

You can [browse the API
documentation](https://migueldeicaza.github.io/SwiftGodot/documentation/swiftgodot/)
but it can also be edited for local use, if you enable it in the
Generator.


# Working with this Repository

You should be all set by referencing this as a package from SwiftPM
but if you want to just work on the binding generator, you will want
to open the Generator project and you can edit the `okList` variable
to trim the build times.

# Driving Godot From Swift

To drive Godot from Swift, use the companion [`SwiftGodotKit`](https://github.com/migueldeicaza/SwiftGodotKit) 
module which embed Godot directly into your application, and
you get to launch the Godot runtime from your code.


# Creating an Extension

Creating an extension that can be used in Godot requires a few 
components:

* Your Swift code: this is where you bring the magic
* A `.gdextension` file that describes where to find the requires
  Swift library assets
* Some Swift registation code and bootstrap code
* Importing your extension into your project

## Your Swift Code

Your Swift code will be compiled into a shared library that Godot
will call.   To get started, the simplest thing to do is to 
create a Swift Library Package that references the Swift Godot 
package, like this:

```
let package = Package(
    name: "MyFirstGame",
    products: [
        .library (name; "MyFirstGame", type: .dynamic, targets: ["MyFirstGame"]),
    ],
    dependencies: [
        .package (url: "https://github.com/migueldeicaza/SwiftGodot")
    ],
    targets: [
        .target (
            name: "MyFirstGame",
            dependencies: ["SwiftGodot"],
            swiftSettings: [.unsafeFlags (["-suppress-warnings"])],
            linkerSettings: [.unsafeFlags (
                ["-Xlinker", "-undefined",
                 "-Xlinker", "dynamic_lookup")]])            
    ]
)
```

The next step is to create your source file with the magic on it,
here we declare a spinning cube:

```
class SpinningCube: Node3D {
    required init () {
        super.init ()
        let meshRender = MeshInstance3D()
        meshRender.mesh = BoxMesh()
        addChild(node: meshRender)
    }

    public override func _process(delta: Double) {
        rotateY(angle: delta)
    }
}
```

Additionally, you need to write some glue code for your 
project to be loadable by Godot, you can do it like this:

```
/// We register our new type when we are told that the scene is being loaded
func setupScene (level: GDExtension.InitializationLevel) {
    if level == .scene {
        register(type: SpinningCube.self)
    }
}

// Export our entry point to Godot:
@_cdecl ("swift_entry_point")
public func swift_entry_point(
    interfacePtr: OpaquePointer?,
    libraryPtr: OpaquePointer?,
    extensionPtr: OpaquePointer?) -> UInt8
{
    print ("SwiftGodot Extension loaded")
    guard let interfacePtr, let libraryPtr, let extensionPtr else {
        print ("Error: some parameters were not provided")
        return 0
    }
    initializeSwiftModule(interfacePtr, libraryPtr, extensionPtr, initHook: setupScene, deInitHook: { x in })
    return 1
}
```

## Bundling Your Extension

To make your extension available to Godot, you will need to 
build the binaries for all of your target platforms, as well
as creating a `.gdextension` file that lists this payload, 
along with the entry point you declared above.

You would create something like this in a file called
`MyFirstGame.gdextension`:

```
[configuration]
entry_symbol = "swift_entry_point"

[libraries]
macos.debug = "res://bin/MyFirstGame"
macos.release = "res://bin/MyFirstGame"
windows.debug.x86_32 = "res://bin/MyFirstGame"
windows.release.x86_32 = "res://bin/MyFirstGame"
windows.debug.x86_64 = "res://bin/MyFirstGame"
windows.release.x86_64 = "res://bin/MyFirstGame"
linux.debug.x86_64 = "res://bin/MyFirstGame"
linux.release.x86_64 = "res://bin/MyFirstGame"
linux.debug.arm64 = "res://bin/MyFirstGame"
linux.release.arm64 = "res://bin/MyFirstGame"
linux.debug.rv64 = "res://bin/MyFirstGame"
linux.release.rv64 = "res://bin/MyFirstGame"
android.debug.x86_64 = "res://bin/MyFirstGame"
android.release.x86_64 = "res://bin/MyFirstGame"
android.debug.arm64 = "res://bin/MyFirstGame"
android.release.arm64 = "res://bin/MyFirstGame"
```

In the example above, the extension always expects the 
platform specific payload to be called "MyFirstGame", 
regarless of the platform.   If you want to distribute
your extension to other users and have a single payload,
you will need to manually set different names for those.

## Installing your Extension

You need to copy both the new `.gdextension` file into 
an existing project, along with the resources it references.

Once it is there, Godot will load it for you.

## Using your Extension

Once you create your extension and have loaded it into
Godot, you can reference it from your code by using the
"Add Child Node" command in Godot (Command-A on MacOS)
and then finding it in the hierarchy.

In our example above, it would appear under Node3D, as it
is a Node3D subclass.
