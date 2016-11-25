# Package Manager Custom Layout and Targets

* Proposal: [SE-XXXX](xxxx-package-manager.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: Discussion

## Introduction

This proposal adds enhancements to `Package.swift` to allow custom layout and targets defination.

## Motivation

The package manager currently uses a convention system to infer various targets like Swift, C family, Tests etc.
This works well for most of the packages but sometimes more flexibility and customization is required.
For example when someone wants to add package manager support to a large project which would be difficult to re-arrange. 
This becomes especially useful when porting an existing C libary to be compatible with package manager, as these libraries have no universal convention. 

## Proposed solution

We propose to extend the `PackageDescription` module (i.e. the manifest file) to allow custom layouts and add additional customization properties to define the targets.

## Detailed Design

### Convention based layouts

We propose to introduce a layout property to `Package` class, which is an enum `PackageLayout`:

```swift
enum PackageLayout {
    /// Use the convention system to infer the targets in the package.
    case automatic(sources: PackageSourcesDirectory)

    /// Turn off package manager's convention system and manually define all targets.
    case custom
}

enum PackageSourcesDirectory {
    /// Automatically infer the sources directory in order: Sources, Src, src, package root.
    case automatic

    /// The sources directory is package root. All valid sources files will be included in target except the manifest file, `Package.swift`.
    case packageRoot

    /// Assume the sources are present at this directory, this must be a relative path from the package root.
    case custom(path: String)
}
```

The `automatic` package layout would work as follows: 
* Use existing conventions to infer the targets in the package.
* The inferred targets still have limited customizable properties like `dependencies`.

Consider the following layout and manifest:

```
├── MySources
│   ├── Bar
│   ├── Baz
│   └── Foo
└── Tests
    └── FooTests
```

```swift
let package = Package(
    name: "MyPackage",
    layout: .automatic(sources: .custom(path: "MySources")),
    targets: [
        Target(
            name: "Foo", 
            dependencies: ["Bar", "Baz"]
        ),
    ]
)
```

Using the existing conventions four targets will be created:

* `Bar`, `Baz`: Swift targets
* `Foo`: Swift target which depends on Bar and Baz
* `FooTests`: Swift test target which depends on `Foo`

### Custom Layouts and Targets

We propose to add the following structures to `PackageDescription` module:

```swift
/// Defines a path for a group of files.
public enum PathSpec {

    /// Include all the files in this directory. Setting recursive to true will also (recusively) search the subdirectories.
    case directory(String, recursive: Bool)

    /// Include the file at this path.
    case file(String)
}

/// Defines the settings of a target.
public struct TargetSettings {

    /// The type of target.
    public enum `Type` {

        /// Swift target.
        case swift

        /// C target which can have headers and a modulemap.
        case c(headers: [PathSpec], modulemap: String)

        /// System module target. The pkgConfig will help package manager search for additional compiler and linker flags associated with the system package.
        case systemModule(pkgConfig: String)
    }

    /// The type of target.
    public let type: Type
    
    /// If the target is a test target.
    public let isTest: Bool
    
    /// If defined, makes the target executable.
    public let executable: Bool
    
    /// The path to sources of the target. This must be relative path from package root.
    public let sources: [PathSpec]
    
    init(type: Type, sources: [PathSpec], isTest: Bool = false, executable: Bool = false) {
        self.type = type
        self.sources = sources
        self.isTest = isTest
        self.executable = executable
    }
}
```

We propose to add an optional property to `Target` class called `settings`. This property will only be allowed (and is compulsory) when layout is set to `.custom`.
What this means is either clients have to use the convention system to construct the package or manage all the targets manually. 
This all or nothing approach is unfortunate but it will help package manager avoid subtle bugs, impossible situations and gotchas.

Package manager will provide runtime errors in case custom layout is used and target settings is not provided or vice versa.

An example of manifest using the custom layout and target settings:

```swift
let package = Package(
    name: "MyPkg",
    layout: .custom,
    targets: [
        Target(
            name: "Foo",
            settings: TargetSettings(
                type: .swift,
                executable: true
                sources: [.directory("Sources/Foo", recursive: true)]
            )
        ),
        Target(
            name: "Bar",
            dependencies: ["Foo"],
            settings: TargetSettings(
                type: .c(
                    headers: [.file("Sources/Foo/unix/foo.h"), .directory("Sources/Foo/include", recursive: true)],
                    modulemap: "Sources/Foo/include/module.modulemap"
                ),
                sources: [.directory("Sources/Foo", recursive: true), .file("Sources/foo.swift")]
            )
        ),
        Target(
            name: "AllTests",
            dependencies: ["Foo", "Bar"],
            settings: TargetSettings(
                type: .swift,
                sources: [.directory("Sources/Tests", recursive: true)],
                isTest: true
            )
        )
    ]
)
```

## Impact on existing code

There will no impact on existing code.

## Alternative considered

None at this point.
