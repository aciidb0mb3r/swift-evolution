# Package Manager Local Dependencies

* Proposal: [SE-NNNN](NNNN-package-manager-local-dependencies.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [TBD](https://github.com/swift-ci)
* Status: **Discussion**

## Introduction

This proposal adds a new API in `PackageDescription` to support declaring
dependency on a package using its path on disk instead of the git URL.

## Motivation

There are two primary motivations for this proposal:

1. Reduce friction in bringing up a set of packages.

	Currently, there is a lot of friction in setting up a new set of interconnected
packages. The packages must be in a git repository and the package author needs
to run the `swift package edit` command for each dependency to develop them in tandem.

2. Some package authors prefer to keep their packages in a single repository.
   This could be because of several reasons, for e.g.:

    * The packages are in very early design phase and are not ready to be published
    in their own repository yet.
    * The author has a set of loosely related packages which they may or may not
    want to publish as a separate repository in future.

## Proposed solution

We propose to add the following API in the `PackageDescription` module:

```swift
extension Package.Dependency {
    public static func package(path: String) -> Package.Dependency
}
```

* This API will only be available if the tools version of the manifest is
  greater than or equal to the Swift version this proposal is implemented in.

* The value of `path` must be a valid absolute or relative path string.

* The package at `path` must be a valid Swift package. 

* The local package will be used as-is and the package manager will not try to
  perform any git operation on the package.

* The local package must be declared in the root package or in other local
  dependencies. This means, it is not possible to depend on a regular versioned
  dependency that declares a local package dependency.

* A local package dependency will override any regular dependency in the package
  graph that has the same package name.

## Impact on existing packages

None.

## Alternatives considered

We considered adding a multi-package repository feature which would allow
authors to publish subpackages from a repository, using git tags with a format
similar to `<package-name/version>`. However, that idea surfaced several
dependency management issues within the repository. We think it is better to
implement a simple solution rather than an overly complicated idea that our
users can't understand or use properly.
