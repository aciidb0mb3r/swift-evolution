# Package Manager Target Layout Spec

* Proposal: [SE-NNNN](NNNN-package-manager-target-layout-spec.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: **Discussion**
* Bug: [SR-29](https://bugs.swift.org/browse/SR-29)

## Introduction

This proposal enhances the `Package.swift` manifest APIs to support custom
target layouts.

## Motivation

The Package Manager uses a convention system to infer targets from disk layout.
This works well for most packages but sometimes more flexibility and
customization is required.  This becomes especially useful when adding Package
Manager support to an existing C library or a large project, which are
difficult to rearrange.

We also think some of the convention rules are confusing and should be removed.

## Proposed solution

* We propose to remove the requirement that name of a test target should have
  suffix "Tests".

* We propose to stop inferring targets from disk. They should be explicitly
  declared in the manifest file. We think that the inference is not much useful
  because it is necessary to declare them in order to use common features like
  product/target dependencies, build settings, etc.

* We propose list of pre-defined search paths for declared targets.

    When an explicit target path is not defined, these directories will be used
    to search for the target. The name of the directory must match the name of
    the target. The search will be done in order and will be case-sensitive.

    Regular targets: package root, Sources, Source, src, srcs.
    Test targets: Tests, package root, Sources, Source, src, srcs.

    It is an error if a target is found in more than one of these paths. In
    such cases, the path should be explicitly declared using the path property
    proposed below.

* We propose to add a factory method `testTarget` to `Target` class to define
  test targets.

    ```swift
    .testTarget(name: "FooTests", dependencies: ["Foo"])
    ```

* We propose to add three properties to `Target` class: `path`, `sources` and
  `exclude`.

    * `path`: This property defines the path to the target, relative to the
      package root. It is not allowed for this path to escape the package root,
      i.e., values like "../Foo", "/Foo" are invalid.  The default value of
      this property will be `nil`, which means the target will be searched in
      the pre-defined paths. Empty string ("") and dot (".") implies package
      root.

    * `sources`: This property defines the source files to be included in the
      target, relative to the target path. The default value of this property
      will be an empty array, which means all valid source files will be
      included.  This can contain both, a directory and an individual source
      file. Directories will be searched recursively for valid source files.

    * `exclude`: This property can be used to exclude certain files and
      directories from being picked up as sources. Exclude paths are relative
      to the target path. This property has more precedence than `sources`
      property.

      _Note: We plan to support glob in future but to keep this proposal short
      we are not proposing it right now._

* It is an error if paths of two targets are overlapping.

    ```swift
    // This is an error:
    .target(name: "Bar", path: "Sources/Bar"),
    .testTarget(name: "BarTests", dependencies: ["Bar"], path: "Sources/Bar/Tests"),

    // This works:
    .target(name: "Bar", path: "Sources/Bar", exclude: ["Tests"]),
    .testTarget(name: "BarTests", dependencies: ["Bar"], path: "Sources/Bar/Tests"),
    ```

* For C family library targets, we propose to add a `publicHeadersPath`
  property.

    This property defines the path to the directory containing public headers
    of a C target. This path is relative to the target path and default value
    of this property is `include`.  
    We think this area can be further improved but there are several behaviors,
    like modulemap generation, which depend of having only one public headers
    directory. We will address those issues separately in a future proposal.

    _All rules related to custom and automatic modulemap remain intact._

* Remove exclude from `Package` class.

    This property is no longer required because of the above proposed
    per-target exclude property.

* The templates provided by the package init subcommand will be updated
  according to the above rules.

## Examples:

* Dummy manifest containing all Swift code.

```swift
let package = Package(
    name: "SwiftyJSON",
    targets: [
        .target(
            name: "Utility",
            path: "Sources/BasicCode"
        ),

        .target(
            name: "SwiftyJSON",
            dependencies: ["Utility"],
            path: "SJ",
            sources: ["SwiftyJSON.swift"]
        ),

        .testTarget(
            name: "AllTests",
            dependencies: ["Utility", "SwiftyJSON"],
            path: "Tests",
            exclude: ["Fixtures"]
        ),
    ]
)
```

* LibYAML

```swift
let packages = Package(
    name: "LibYAML",
    targets: [
        .target(
            name: "libyaml",
            sources: ["src"]
        )
    ]
)
```

* Node.js http-parser

```swift
let packages = Package(
    name: "http-parser",
    targets: [
        .target(
            name: "http-parser",
            publicHeaders: ".",
            sources: ["http_parser.c"]
        )
    ]
)
```

* swift-build-tool

```swift
let packages = Package(
    name: "llbuild",
    targets: [
        .target(
            name: "swift-build-tool",
            path: ".",
            sources: [
                "lib/Basic",
                "lib/llvm/Support",
                "lib/Core",
                "lib/BuildSystem",
                "products/swift-build-tool/swift-build-tool.cpp",
            ]
        )
    ]
)
```

## Impact on existing code

These enhancements will be added to the version 4 manifest API which will
release with Swift 4. There will be no impact on packages using the version 3
manifest API.  When packages update their minimum tools version to 4.0, they
will need to update the manifest according to the changes in this proposal.

There are two flat layouts supporting right now:

1. Source files under package root.
2. Source files under `Sources/` directory.

If packages want to continue using these layouts, they will need to explicitly
provide path to the targets. For example, if a package `Foo` has following
layout:

```
Package.swift
Sources/main.swift
Sources/foo.swift
```

The updated manifest will look like this:

```swift
// swift-tools-version:4.0
import PackageDescription

let package = Package(
    name: "Foo",
    targets: [
        .target(name: "Foo", path: "Sources"),
    ]
)
```

## Alternatives considered

None.
