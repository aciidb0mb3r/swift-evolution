# Package Manager Extensible Build Tools

* Proposal: [SE-NNNN](NNNN-package-manager-extensible-tools.md)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r), [Daniel Dunbar](https://github.com/ddunbar)
* Review Manager: [TBD](https://github.com/)
* Status: **WIP**

## Introduction

This is a proposal for adding package manager support for extensible build
tools, i.e. executing tools at build time which were themselves produced by some
other package, and whose behavior the package manager does not understood
directly, but rather interacts through a well-defined, Swift, protocol.

We expect this behavior to greatly enhance our capacity for building complex
software packages.

## Motivation

There are large bodies of existing, complex, software projects which cannot be
described directly in SwiftPM, but which other projects wish to depend upon.

The package manager currently supports two mechanisms by which non-package
manager projects can be used in other packages:

* The system module map feature allows defining support for a package which
  already exists in an installed form on the system. In conjunction with system
  package managers, this can be used to configure an environment in which
  a package can depend on such a body of software.

* The C language targets feature allows the package manager to build and include
  C family targets. This can provide similar features to the previous bullet,
  but through use of the standard C compiler features (like header and library
  search paths). The external project again needs to be installed using a
  different package manager.

These mechanisms have two major downsides:

1. The usability of a package depending upon them requires interaction with a
   system package manager. This increases developer friction.

2. The system packages are global. This means project builds are no longer self
   contained. That introduces greater complexity when managing the dependencies
   of multiple projects which need to run in the same environment, but may have
   conflicting dependencies. This is compounded by the package manager itself
   not understanding much about the system dependencies.

The package manager also currently has no support for running third-party tools
during the build process. For example, the [Swift protobuf compiler](https://github.com/apple/swift-protobuf) is used to generate 
Swift code automatically for models defined in ancillary sources. Currently
there is no way for a package to directly cause this compiler to be run during
the build, rather the user must manually invoke the compiler before invoking the
package manager.

## Proposed solution

We will introduce a new type of target and product called "Package Extension".
A Package Extension target should contain non-executable Swift source code,
which will be compiled into a dynamic library. This target will have access to
a new runtime module called `PackageExtension`.

A Package Extension target should declare dependency on all executable products
that it needs. The executable products can be in the same package (as the
package extension) or they can be an executable product in one of the package
dependency. It is not required to declare an executable as a product if the
executable is in the same package.

In order to allow other packages to depend on a Package Extension, it must be
exported using the new Package Extension product type. This export is not
necessary if the Package Extension is used within the same package.

Initially, only executables will be allowed as tools to create build commands,
but we do plan on adding support for defining in-process build tools. This is
dependent on SwiftPM adopting llbuild's C API, which is a very desirable goal.
Similarly, we will not allow a Package Extension target to depend on a library
target or product until we add the support for in-process build tools.

We will start with a very strict and minimal API for the new `PackageExtension`
module and evolve as we discover new requirements. The process for evolving the
API will be same as that of the `Package.swift` API, i.e., we will use the Swift
Evolution process.

The API in `PackageExtension` module will be tied to the Swift Tools Version
declared in the manifest of the package that extension is in. This means, to use
an API that was added in Swift version `X.Y.Z`, the tools version of the package
should be at least `X.Y.Z`.


## Detailed design

To allow declaring Package Extension targets and products, we will add the
following API in the `Package.swift` manifest:

```swift
extension Target {
    static func packageExtension(
        name: String,
        dependencies: [Dependency] = []
    ) -> Target
}

extension Product {
    static func packageExtension(
        name: String
    ) -> Product
}
```

We will add a new array parameter `buildRules` to regular and test target types
to allow declaring custom build rules. The initial API is described below:

```swift
final class BuildRule {
    /// The source files in this build rule.
    ///
    /// This is an array of glob patterns used to identify the input files for this
    /// build rule.
    ///
    /// Paths specified must be relative to the target path.
    var sources: [String]

    /// The package extension, this build rule is defined in.
    // FIXME: Should we allow package extensions to declare more than one build
    // rule? If so, we should add another parameter for declaring the build rule
    // name in addition to the package extension.
    var packageExtension: String

    /// The options dictionary that will be available to this build rule.
    var options: [String: Any]

    /// Create a new build rule.
    static func build(
        sources: [String],
        withPackageExtension PackageExtension: String,
        options: [String: Any]
    )
}
```

We propose the following API for the initial version of `PackageExtension`
runtime. These APIs are **not** final and will probably need some refinement
once we try some community build tools with an actual implementation of this
proposal. However, we hope the refinements will be minimal and will not require
another round of review.

```swift
/// Describes a custom build rule.
///
/// Package extensions must implement this protocol and create 
/// an instance using the convention described below. Currently, 
/// there can be only one build rule in a package extension.
///
///     @_cdecl("createCustomBuildRule")
///     func createCustomBuildRule() -> Any {
///         return MyCustomBuildRule()
///     }
protocol CustomBuildRule {

    /// Called to construct tasks.
    func constructTasks(target: TargetBuildContext, delegate: TaskGenerationDelegate) throws
}

/// Describes the context in which a target is being built.
protocol TargetBuildContext {

    /// The name of the target being built.
    var targetName: String { get }

    /// The inputs to this target.
    var inputs: [Path] { get }

    /// The build directory for the target.
    ///
    /// Custom build rules are not allowed to produce outputs
    /// outside of this directory.
    var buildDirectory: Path { get }

    /// The custom options defined in the manifest file for this target.
    var options: [String: Any] { get }

    /// Finds the given tool.
    func lookup(tool: String) throws -> Tool
}

/// Interface used to generate create custom tasks by a build rule.
protocol TaskGenerationDelegate {

    /// Creates a command which will be executed as part of the build process.
    ///
    /// The tool and node instance must be created by the respective APIs.
    /// Custom implementations will be rejected at runtime.
    func createCommand(tool: Tool, inputs: [Node], outputs: [Node])

    /// Creates a node for the given path.
    func createNode(_ path: Path) -> Node

    /// Adds a derived source file, which will be input to other build rules.
    func addDerivedSource(_ path: Path)

    /// Returns the diagnostics engine used for emitting diagnostics.
    var diagnostics: DiagnosticsEngine { get }
}

/// Represents a build tool.
///
/// The tools can be looked up from the build context.
///
/// Currently, a tool must be an executable dependency
/// to this package extension.
protocol Tool {}

/// Represents a build node.
///
/// Nodes should only be created using the task generation delegate.
protocol Node {}

/// Represents an absolute path on disk.
// FIXME: Should this be a struct instead?
protocol Path {

    /// The string value of the path.
    var string: String { get }

    /// Returns the basename of the path.
    var basename: String { get }

    /// Creates a new path by appending the given subpath.
    func appending(_ subpath: String) -> Path
}

/// An engine for managing diagnostic output.
protocol DiagnosticsEngine {

    /// Emits the given error.
    ///
    /// Note: Emitting an error will abort the build process.
    func emit(error: String)

    /// Emits the given warning.
    func emit(warning: String)

    /// Emits the given note.
    func emit(note: String)
}
```

## Example

Consider an example version of the `swift-protobuf` package:

```sh
Protobuf:
  PBLib/
  PBTool/
  PBPackageExt/
```

#### `Package.swift`

```swift
let package = Package(
    name: "Protobuf",
    products: [
        .packageExtension(name: "PBPackageExt"),
    ],
    targets: [
        .target(
            name: "PBLib",
            dependencies: []),
        .target(
            name: "PBTool",
            dependencies: ["Lib"]),
        .packageExtension(
            name: "PBPackageExt",
            dependencies: ["PBTool"]),
    ]
)
```

#### `PackageExtension.swift`:

```swift
import PackageExtension

struct ProtobufBuildRule: CustomBuildRule {

    func construct(target: TargetBuildContext, delegate: TaskGenerationDelegate) throws {

        // Create a command for each input file.
        for inputFile in target.inputs {

            // Compute the output file.
            let outputFile = buildDirectory.appending("DerivedSources/\(inputFile.basename)")

            // Construct the command line.
            var commandLine: [String] = []

            // Add the input file.
            commandLine += "-c" + inputFile.string

            if case let extraFlags as [String] = target.options["OTHER_FLAGS"] {
                // Append any extra flags as-is.
                commandLine += extraFlags

                // Inform `-v` is deprecated.
                if context.options.contains("-v") {
                    delegate.diagnostics.emit(warning: "-v is deprecated; use --verbose instead")
                }
            }

            // Add the output information.
            commandLine += ["-o", outputFile.string]

            // Create the command to build the swift source file.
            delegate.createCommand(
                tool: try target.lookup(tool: "PBTool"),
                inputs: [delegate.createNode(inputFile)],
                outputs: [delegate.createNode(outputFile)],
                description: "Generating Swift source for \(inputFile.string)"
            )

            // Add the output file as a derived source file.
            delegate.target.addDerivedSource(outputFile)
        }
    }
}

@_cdecl("createCustomBuildRule")
func createCustomBuildRule() -> Any {
    return ProtobufBuildRule()
}
```

#### My package

```sh
MyPkg:
 Tool/
   main.swift
   misc.proto
   ADT/
        Lock.proto
        Queue.proto
```

Package.swift:

```swift
let package = Package(
    name: "MyPkg",
    dependencies: [
        .package(url: "https://github.com/Utilities/SwiftyCURL", from: "1.0.0"),
        .package(url: "https://github.com/apple/swift-protobuf", from: "1.0.0"),
    ],
    targets: [
        .target(
            name: "Tool",
            dependencies: ["SwiftyCURL"],
            customRules: [
                .build(
                    sources: ["misc.proto", "ADT/*.proto"]
                    withPackageExtension: "PBPackageExt",
                    options: [
                        "OTHER_FLAGS": ["-emit-debug-info", "-warnings-as-errors", "-v"],
                    ],
                ),
            ],
        )
    ]
)
```

## Alternatives considered

We considered allowing a more straight-forward capability for the package
manager to simply run "shell scripts" (essentially arbitrary invoke command
lines) at points during the build. We rejected this approach because:

1. Even this approach requires us to either explicitly or implicit document and
   commit to supporting a specific file system layout for the build artifacts
   (so that they scripts can interact with them). Adding support in this way
   makes it hard for script authors to know what is explicitly officially
   supported and what simply happens to work due to the current implementation
   details of the tool. That in turn could make it hard to evolve the tool if we
   wanted to change a behavior which numerous scripts had grown to depend on.

2. It is hard for us to enforce that scripts don't do things that are
   unsupported, since the script by design is intended to interact directly with
   the file system. This has similar problems as #1 and makes it harder for
   package authors to write "correct" packages.


Another alternative is to do nothing, and requiring all behaviors be explicitly
supported through some well-modeled behavior defined in the package
manifest. While we aim to support as many common behaviors as possible, we also
want to support as many complex behaviors as possible and recognize that we need
an extensible mechanism to support "special cases". Our intention is that even
once we add extensible build tools, that we will continue to add explicit
manifest support for behaviors that we see become common in the ecosystem, in
order to keep packages simple to understand and author.



Although it is possible to the more straightforward "shell script" capability is
simpler to implement and could be added to the existing package manager without
significant implementation work, we have felt that this would ultimately be more
likely to harm than help the ecosystem. We have several reasons for believing
this:

1. We know the package manager is currently missing critical features which
   would be needed by many packages. One of our design tenants has been that we
   should design so that roughly 80% of packages can be written with a
   _straightforward_, _simple_, and _clean_ manifest that does not require
   advanced features. If we were to add a straightforward, but complex, script
   based extension mechanism, we expect that far too many packages would begin
   to take advantage of it due to these missing features. Due to the opaque
   nature of shell-script extensions, this would be very hard to then migrate
   past once we did gain the appropriate features, because the tools would be in
   a poor position to understand what the shell script did.

2. The package manager currently always builds individual packages into discrete
   sandboxes, including the transitive closure of the package
   dependencies. While this works well for sandboxing effects, it is inherently
   not scalable when many separate packages are being worked on by the same
   developer. This approach also makes it hard for continuous integration
   systems which need to perform very reliable tests on many different packages.

   Our intention is to solve these problems by leveraging
   [reproducible build](https://reproducible-builds.org) techniques to expose
   the same user interface as we do today, but transparently cache shared build
   artifacts under the hood. This will rely on the ability of the package
   manager to have perfect knowledge of exactly what content is used by a
   particular part of a build. Shell script based hook mechanisms make this very
   difficult, since (a) by their nature they pull in a large number of
   dependencies (the shell, the tools used in the shell script, etc.), and (b)
   it is hard for the tool to reason about them.
