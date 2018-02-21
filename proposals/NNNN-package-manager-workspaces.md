# Package Manager Workspace

* Proposal: [SE-NNNN](NNNN-package-manager-workspace.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [TBD](https://github.com/swift-ci)
* Status: **Discussion**

## Introduction

This proposal introduces a new feature called "workspace" to help Swift package
authors to develop packages that depend on each other.

## Motivation

Swift package authors usually want to develop their set of packages in tandem.
This is currently possible using the `swift package edit` feature.  However,
that feature is suitable for one-off edits rather than continued development.
Moreover, edited packages are not persistent and thus can't be shared with the
development team.

Package authors also want to know the impact on downstream dependencies before
committing a change. There is no obvious way to do this right now but there are
some options like creating a pre-release tag or editing the package in the
downstream dependency. Both of these are suboptimal solutions.

## Proposed solution

We propose a new feature in package manager called "workspace". A SwiftPM
workspace will be driven by a new manifest file called `Workspace.swift`. The
file will contain a list of packages and each package will act as a root
package. The dependency resolver will combine dependency requirements from each
root package before resolving them. This means that it is possible that
a certain root package is overly constraining a dependency requirement.

## Detailed design

* Initially, we will introduce the following APIs for `Workspace.swift` file:

```swift
/// Defines a package workspace.
final class Workspace {

    /// Represents a package inside a workspace.
    final class Package {

        /// The URL of the package.
        ///
        /// This will be used to clone the package at `path` if it doesn't exist.
        let url: String?

        /// The local path of the package.
        let path: String

        /// Any custom user data associated with the package.
        let userInfo: JSON?

        private init(
            url: String?,
            path: String,
            userInfo: JSON?
        ) {
            self.url = url
            self.path = path
            self.userInfo = userInfo
        }

        static func package(
            url: String? = nil,
            path: String,
            userInfo: JSON? = nil
        ) -> Package {
            return Package(url: url, path: path, userInfo: userInfo)
        }
    }

    /// The packages in the workspace.
    var packages: [Package]

    init(packages: [Package]) {
        self.packages = packages
    }
}
```

### Example:
 
```swift
// swift-tools-version:4.2
import PackageDescription

let workspace = Workspace(packages: [

    .package(
        url: "https://github.com/apple/example-package-dealer",
        path: "../dealer",
        userInfo: .init(["branch": "master"])),

    .package(
        url: "https://github.com/apple/example-package-fisheryates",
        path: "../fisheryates"),

    .package(
        url: "https://github.com/apple/example-package-deckofplayingcards",
        path: "../deckofplayingcards"),

    .package(
        path: "../playingcard"
        userInfo: .init(["fork": "https://github.com/aciidb0mb3r/example-package-playingcard"])),
])
```

* The `url` property is optional and will be used to clone the root package if it
  does not exist at `path`.

* The `userInfo` property allows associating arbitary data with a package. The
  package manager will not read its content and will dump it as-is when JSON
  representation of the workspace manifest is asked.

* If some root package is also a dependency in the overall package graph, the
  dependency requirement will be ignored and the root package will be used
  instead.

* The package manager will build all root packages in the same build directory
  which is controlled by the `--build-path` option.

* The package manager will write a file called `Workspace.resolved` 
  to record the dependency resolution result. The behaviour of this file will be
  completely similar to the existing `Package.resolved` file. Generally, this
  means, if `Workspace.resolved` is present, the package manager will
  prefer the recorded versions.

* It will not be allowed to have both `Package.swift` and `Workspace.swift`
  in the same directory. We may lift this restriction in future by adding an
  option to prefer one over the other.

* Package authors can check-in this file in a repository and share the workspace
  with a development team. Note that this file can be used to create different
  workspaces for automated testing of downstream dependencies. 

## Impact on existing packages

None.

## Alternatives considered

None.
