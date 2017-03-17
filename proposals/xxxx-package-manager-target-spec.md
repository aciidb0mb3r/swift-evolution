# Package Manager Custom Layout and Targets


* Proposal: [SE-XXXX](xxxx-package-manager.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: Discussion

## Introduction

This proposal adds enhancements to `Package.swift` to allow custom source layouts.

## Motivation

The package manager uses a convention system to infer various targets like
Swift, C family, tests etc.  This works well for most packages but sometimes
more flexibility and customization is required.  
For example, when someone wants to add package manager support to a large
project which would be difficult to re-arrange.  This becomes especially useful
when porting an existing C libary to be compatible with package manager, as
these libraries have no universal convention. 

## Proposed solution

### Targets Directory:

We propose a "Targets Directory", each subdirectory  of the targets directory
will define a target. It is invalid to place any source file under this
directory.

We propose to add an enum `TargetDirectory` to define the path of targets
directory.

```swift
enum TargetDirectory {
    /// Use the default location for targets directory i.e. Sources.
    case `default`

    /// Define a custom path to the targets directory.
    ///
    /// This must be a relative path from the package root.
    case custom(path: String)
}
```

By default, the value of this enum will be set to `default`.

Example:

Layout:

```
├── MySources
│   ├── Bar
│   ├── Baz
│   └── Foo
└── Tests
    └── FooTests
```

Manifest:

```swift
let package = Package(
    name: "MyPackage",
    targetsDirectory: .custom(path: "MySources"),
    targets: [
        .target(
            name: "Foo", 
            dependencies: ["Bar", "Baz"]
        ),
    ]
)
```

In this example, assuming all targets contains Swift sources, four targets will
be created:

* `Bar`, `Baz`: Swift targets
* `Foo`: Swift target which depends on `Bar` and `Baz`
* `FooTests`: Swift test target which depends on `Foo`

### Custom Layouts:

We propose to add the following properties to `Target` class:

```swift
final class Target {
    
    /// Path to the modulemap file. Only valid for C family library targets.
    var moduleMap: String?

    /// Look for the sources defined by these patterns.
    var sources: [String]

    /// The header search paths. Only valid for C family library targets.
    var headers: [String]
    
    /// The excluded sources patterns. 
    ///
    /// These have higher preference than `sources` property, i.e. if some file
    /// is picked by both sources and excludes pattern, it will be excluded.
    var excludes: [String]

    /// Explicity mark the target as executable or a library. Package manager
    /// will try to guess the value based on presence of `main.extension` file.
    ///
    /// Note: It is required for Swift executable targets to have a 'main.swift'
    /// file containing top-level code.
    var executable: Bool?
}
```

The following pattern will be supported by `headers`, `sources` and `excludes`:

| Pattern  | Description                                    |
| -------- |:-----------------------------------------------| 
| *        | matches all files in the directory             |
| **       | recursively matches all files in the directory | 


For example, consider the following structure:

```
└── somedir
    ├── bar.swift
    ├── baz.swift
    └── foo
        └── baz.swift
```

| Pattern             | Result                                                      |
| ------------------- |:------------------------------------------------------------| 
| `somedir/bar.swift` | somedir/bar.swift                                           |
| `somedir/*`         | somedir/bar.swift, somedir/baz.swift                        |
| `somedir/foo/*`     | somedir/foo/baz.swift                                       |
| `somedir/**`        | somedir/bar.swift, somedir/baz.swift, somedir/foo/baz.swift |

_Note: We might consider extending this pattern to match unix glob in future._

### Examples:

These would be manifest of some C packages without changing their file
structure or source files:

* LibYAML

```swift
let packages = Package(
    name: "LibYAML",
    targets: [
        Target(
            name: "libyaml",
            sources: [
                "src/*",
            ],
            // Dummy property.
            defines: [
                "YAML_VERSION_MAJOR=0",
                "YAML_VERSION_MINOR=1",
                "YAML_VERSION_PATCH=7",
                "YAML_VERSION_STRING=\"0.1.7\"",
            ]
        )
    ]
)
```

* Node.js http-parser

```swift
let packages = Package(
    name: "http-parser",
    targets: [
        Target(
            name: "http-parser",
            moduleMap: "module.modulemap",
            sources: [
                "http_parser.c",
            ]
        )
    ]
)
```

```C
module httpparser {
    header "http_parser.h"
    export *
}
```

* libressl

```swift
let packages = Package(
    name: "libressl",
    targets: [
        Target(
            name: "libressl",
            moduleMap: "include/module.modulemap",
            headerPaths: [
                "include",
                "include/compat",
                "crypto",
                "crypto/evp",
                "crypto/asn1",
                "crypto/modes",
            ],
            sources: [
                "ssl/*",
                "tls/*",
                "crypto/**",
            ],
            exclude: [
                /* Exclude windows files */
                "crypto/bio/b_win.c",
                "crypto/ui/ui_openssl_win.c",
                "crypto/des/ncbc_enc.c",
                "crypto/poly1305/poly1305-donna.c",

                /* Exclude compat files already present on macOS. */
                "crypto/compat/bsd-asprintf.c",
                "crypto/compat/explicit_bzero_win.c",
                "crypto/compat/inet_pton.c",
                "crypto/compat/posix_win.c",
                "crypto/compat/strlcat.c",
                "crypto/compat/strlcpy.c",
                "crypto/compat/getentropy*",
            ],
            // DUMMY.
            defines: [
                "OPENSSL_NO_HW_PADLOCK",
                "HAVE_STRCASECMP",
                "HAVE_STRLCPY",
                "HAVE_STRLCAT",
                "HAVE_STRNDUP",
                "HAVE_STRNLEN",
                "HAVE_STRSEP",
                "HAVE_TIMINGSAFE_BCMP",
                "HAVE_MEMMEM",
                "LIBRESSL_INTERNAL",
            ]
        )
    ]
)
```

```C
module libressl {
    exclude header "openssl/cms.h"
    umbrella header "openssl/openssl.h"
    export *
    link "Clibressl"
}
```

* swift-build-tool

```swift
let packages = Package(
    name: "llbuild",
    targets: [
        Target(
            name: "swift-build-tool",
            sources: [
                "lib/Basic/*",
                "lib/llvm/Support/*",
                "lib/Core/*",
                "lib/BuildSystem/*",
                "products/swift-build-tool/swift-build-tool.cpp",
            ],
            executable: true
        )
    ]
)
```

## Impact on existing code

// TODO

## Alternative considered

None at this point.
