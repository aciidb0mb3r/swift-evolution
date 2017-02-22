# Package Manager Manifest API Redesign

* Proposal: [SE-XXXX](xxxx-package-manager-manifest-api-redesign.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: **Discussion**

## Introduction

This is a proposal for redesigning the `Package.swift` manifest APIs provided by Swift Package Manager.  
This proposal does not aim to change any actual underlying functionality.

## Motivation

The `Package.swift` manifest APIs were designed prior to the [API Design Guidelines]
(https://swift.org/documentation/api-design-guidelines/), and their design was not reviewed by the
evolution process. Additionally, there are several small areas which can be cleaned up to make
the overall API more "Swifty".

We would like to redesign these APIs as necessary to provide clean, conventions-compliant APIs
that we can rely on in the future.

## Proposed solution

Note: Access modifier is omitted from diffs and examples for brevity. The access modifier is `public` for
all APIs.

* Remove `successor()` and `predecessor()` from `Version`.

    These methods neither have well defined semantics nor are used a lot. For e.g., the current
    implementation of `successor()` always just increases the patch version.


    <details>
      <summary>View diff</summary>
      <p>
    ```diff
    struct Version {
    -    func successor() -> Version

    -    func predecessor() -> Version
    }
    ```
    </p></details>

* Make all properties of `Package` and `Target` mutable.

    Currently, `Package` has three immutable and four mutable properties, and `Target` has one
    immutable and one mutable property. We propose to make all properties mutable to allow
    conditional compilation behavior; at least until the package manager gains explicit support
    for conditionals.

    <details>
      <summary>View diff and example</summary>
      <p>

      Diff:
    ```diff
    final class Target {
    -    let name: String
    +    var name: String
    }

    final class Package {
    -    let name: String
    +    var name: String

    -    let pkgConfig: String?
    +    var pkgConfig: String?

    -    let providers: [SystemPackageProvider]?
    +    var providers: [SystemPackageProvider]?
    }
    ```

    Example:
    ```swift
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(name: "Foo", dependencies: ["Bar"]),
        ]
    )

    #if os(Linux)
    package.targets[0].dependencies = ["BarLinux"]
    #endif
    ```
    </p></details>

* Change `Target.Dependency` enum cases to lowerCamelCase.

    According to API design guidelines, everything other than types should be lowerCamelCase.

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    enum Dependency {
    -    case Target(name: String)
    +    case target(name: String)

    -    case Product(name: String, package: String?)
    +    case product(name: String, package: String?)

    -    case ByName(name: String)
    +    case byName(name: String)
    }
    ```

    Example:
    ```diff
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(
                name: "Foo", 
                dependencies: [
    -                .Target(name: "Bar"),
    +                .target(name: "Bar"),

    -                .Product(name: "SwiftyJSON", package: "SwiftyJSON"),
    +                .product(name: "SwiftyJSON", package: "SwiftyJSON"),
                ]
            ),
        ]
    )
    ```
    </p></details>

* Add default parameter to the enum case `Target.Dependency.product`.

    The associated value `package` in the (enum) case `product`, is an optional `String`. It should
    have the default value `nil` so clients don't need to write it if they prefer using
    explicit enum cases but don't want to specify the package name i.e. it should 
    be possible to write `.product(name: "Foo")` instead of `.product(name: "Foo", package: nil)`.

    If [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md)
    is accepted, we can directly add a default value. Otherwise, we will use a static
    factory method to provide default value for `package`.

* Upgrade `SystemPackageProvider` enum to a struct.

    This enum allows SwiftPM System Packages to emit hints in case of build failures due to
    absence of a System Package. Only one System Package per System Packager can be specified
    currently. We propose to allow specifying multiple System Packages by replacing the enum 
    with this struct:

    ```swift
    public struct SystemPackageProvider {
        enum PackageManager {
            case apt
            case brew
        }
    
        /// The system package manager.
        let packageManager: PackageManager 

        /// The array of system packages.
        let packages: [String]
    
        init(_ packageManager: PackageManager, packages: [String])
    }
    ```

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    -enum SystemPackageProvider {
    -    case Brew(String)
    -    case Apt(String)
    -}

    +struct SystemPackageProvider {
    +    enum PackageManager {
    +        case apt
    +        case brew
    +    }
    +
    +    /// The system package manager.
    +    let packageManager: PackageManager 
    +
    +    /// The array of system packages.
    +    let packages: [String]
    +
    +    init(_ packageManager: PackageManager, packages: [String])
    +}
    ```

    Example:

    ```diff
    let package = Package(
        name: "Copenssl",
        pkgConfig: "openssl",
        providers: [
    -        .Brew("openssl"),
    +        SystemPackageProvider(.brew, packages: ["openssl"]),

    -        .Apt("openssl-dev"),
    +        SystemPackageProvider(.apt, packages: ["openssl", "libssl-dev"]),
        ]
    )
    ```
    </p></details>


* Remove implicit target dependency rule for test targets.

    There is an implicit test target dependency rule: a test target "FooTests"
    implicity depends on a target "Foo", if "Foo" exists and "FooTests" doesn't explicitly
    declare any dependency. We propose to remove this rule because:

    1. It is a non obvious "magic" rule that has to be learned.
    2. It is not possible for "FooTests" to remove dependency on "Foo" while having no other (target) dependency.
    3. It makes real dependencies less discoverable.
    4. It may cause issues when we get support for mechanically editing target dependencies.

* Introduce an "identity rule" to determine if an API should use an initializer or a factory method:

    Under this rule, an entity having an identity, will use a type initializer
    and rest will use factory methods. `Package`, `Target` and `Product` are
    identities. However, a product referenced in a target dependency is not an
    identity.

    This means the `Product` enum should be converted into an entity. We propose
    to introduce a `Product` class with two subclasses: `Executable` and `Library`.
    They will be defined as follow:

    ```swift
    class Product {
        
        let name: String
        let targets: [String]

        fileprivate init(name: String, targets: [String])
    }

    final class Executable: Product {
        init(name: String, targets: [String])
    }

    final class Library: Product {
        enum LibraryType: String {
            case `static`
            case `dynamic`
        }

        let type: LibraryType?
         
        init(name: String, type: LibraryType? = nil, targets: [String])
    }
    ```

    <details>
      <summary>View example</summary>
      <p>

    Example:

    ```swift
    let package = Package(
        name: "Foo",
        target: [
            Target(name: "Foo", dependencies: ["Utility"]),
            Target(name: "tool", dependencies: ["Foo"]),
        ],
        products: [
            Executable(name: "tool", targets: ["tool"]), 
            Library(name: "Foo", targets: ["Foo"]), 
            Library(name: "FooDy", type: .dynamic, targets: ["Foo"]), 
        ]
    )
    ```
    </p></details>

* Special syntax for version initializers.

    A simplified summary of what is supported in other package managers:

    | PM | x-ranges | ~ | ^ |
    |---|---|---| --- |
    | npm | Supported | Allows patch-level changes if a minor version is specified on the comparator. Allows minor-level changes if not.  | patch and minor updates |
    | Cargo | Supported | Same as above | Same as above |
    | CocoaPods | Not supported | Same as above | Not present |
    | Carthage | Not supported | patch and minor updates| Not present |

    * Every package manager supports the tilde `~` operator in some form.
    * Most people never understand `~`.
    * The widely accepted suggestion on how to constrain your versions is "use
      `~>`, it does the right thing".
    * SwiftPM will probably have more novice users, because it comes built-in.
    * A lot of people even explicitly set a concrete version simply because they
      don't know better (Proposal author is guilty of that mistake).

    x-range seems to be easy to understand because of its clarity. Some e.g.:

    1. 1.2.x means 1.2.0 ..< 1.3.0
    2. 1.x.x or 1.x means 1.0.0 ..< 2.0.0

    Maybe something like this:
    ```swift
    .package(url: "/SwiftyJSON", "1.x", greaterThan: "1.2.3"),
    .package(url: "/SwiftyJSON", "1.x", .greaterThan("1.2.3")),
    .package(url: "/SwiftyJSON", "1.x", >"1.2.3"),

    // Constraint to an exact version.
    .package(url: "/SwiftyJSON", "1.2.3"),

    // Constraint to an arbitrary range.
    .package(url: "/SwiftyJSON", "1.2.3"..<"1.3.0"),

    // Constraint to an arbitrary closed range.
    .package(url: "/SwiftyJSON", "0.0.3"..."0.0.8"),
    ```

    Current possible stuff:
    ```swift
    // Constraint to a major version.
    .package(url: "/SwiftyJSON", majorVersion: 1),

    // Constraint to a major and minor version.
    .package(url: "/SwiftyJSON", majorVersion: 1, minor: 2),

    // Constraint to an exact version.
    .package(url: "/SwiftyJSON", "1.2.3"),

    // Constraint to an arbitrary range.
    .package(url: "/SwiftyJSON", versions: "1.2.3"..<"1.2.6"),

    // Constraint to an arbitrary closed range.
    .package(url: "/SwiftyJSON", versions: "1.2.3"..."1.2.8"),
    ```

    **// FIXME: Need to Discuss.**

* Eliminate exclude.

    It is bad at top-level and an awkward wart on the Package that
    we would prefer be replaced by real package convention definition. 

    **// FIXME: Need to Discuss.**

* Adjust order of parameters on `Package` class:

    We propose to reorder the parameters of `Package` class to: `name`,
    `pkgConfig`, `products`, `dependencies`, `targets`, `compatibleSwiftVersions`.

    The rational behind this reorder is that the most interesting parts of a
    package are its product and dependencies. Targets are usually important
    during development of the package.  Placing them at the end makes it easier
    for the developer to jump to end of the file to access them.


    <details>
      <summary>View example</summary>
      <p>

    Example:

    ```swift
    let package = Package(
        name: "Paper",
        products: [
            Executable(name: "tool", targets: ["tool"]),
            Libary(name: "Paper", type: .static, targets: ["Paper"]),
            Libary(name: "PaperDy", type: .dynamic, targets: ["Paper"]),
        ],
        dependencies: [
            .package(url: "http://github.com/SwiftyJSON", "1.x"),
            .package(url: "../CHTTPParser", "2.x", greaterThan: "2.1.0"),
            .package(url: "http://some/other/lib", "1.2.3"),
        ]
        targets: [
            // Helper tool.
            Target(
                name: "tool",
                dependencies: [
                    "Paper",
                    "SwiftyJSON"
                ]),
            // The main library.
            Target(
                name: "Paper",
                dependencies: [
                    "Basic",
                    .target(name: "Utility"),
                    .product(name: "CHTTPParser"),
                ])
        ]
    )
    ```
    </p></details>

    **//FIXME: Need to update after we decide on version spec**

## Impact on existing code

The above changes will be implemented only in the new Package Description v4
library. The v4 runtime library will release with Swift 4 and packages will be
able to opt-in into it as described by
[SE-152](https://github.com/apple/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md).

## Alternatives considered

* Add variadic overloads.

    Adding variadic overload allows omitting parenthesis which leads to less cognitive load
    on eyes, especially when there is only one value which needs to be specified. For e.g.:

        Target(name: "Foo", dependencies: "Bar")

    looks better than:

        Target(name: "Foo", dependencies: ["Bar"])

    However, plurals words like `dependencies` and `targets` imply a collection which implies
    brackets. It also makes the grammar wrong. Therefore, we reject this option.
    
* Version exclusion.
    
    It is not uncommon to have a specific package version break something, and
    it is undesirable to "fix" this by adjusting the range to exclude it
    because this overly constrains the graph and can prevent picking up the
    version with the fix.

    This is desirable but it should be proposed separately.

* Inline package declaration.

    We should probably support declaring a package dependency anywhere we support
    spelling a package name. It is very common to only have one target require
    a dependency, and annoying to have to specify the name twice.

    This is desirable but it should be proposed separately.
