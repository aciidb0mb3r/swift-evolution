# Package Manager Multi-Package Repositories

* Proposal: [SE-NNNN](NNN-package-manager-multi-package-repo.md)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r), [Daniel Dunbar](https://github.com/ddunbar)
* Review Manager: [TBD](https://github.com/)
* Status: Discussion

## Introduction

The package manager currently expects each repository to contain a single
package. This is a proposal for adding package manager support for repositories
which contain multiple packages.

## Motivation

Large teams and organizations often have many projects coexisting in one
repository. In some organizations, this can even take the form of _all_ projects
being in a single repository (a "monorepo"). For other organizations, for
example LLVM + Clang + Swift itself, this may take a much more limited scope of
related projects split among repositories in an ad hoc manner.

Adoption of the Swift package manager in these environents requires some ability
for packages to (a) not be tied to being in the root of the repository, and (b)
be able to co-exist in the same repository, without forcing everything to be in
one package.

## Proposed solution

We propose to allow arbitrary organization of packages within a repository,
which we will identify as packages by their declaration in a file called
`Packages.json`.

A package in a multi-package repository will generally have the same package
features as if it was in its own repository. 

Each package in the repository will be required to have a unique name defined by
the package name in the manifest. This will be the name used to refer to the
package in other dependency specifications. Furthermore, each package will
have its own set of versions.

When performing dependency resolution, it will be possible for the resolver to
select different versions for packages in the same repository.

## Terminology

**Top-level package**: A package present at root of a repository.

**Multi-package repository**: A repository containing more than one package.
A repository is a multi-package repository if it contains the file
`Packages.json` at the repository root.

## Detailed design

We propose the following behaviour in a multi-package repository:

The presence of the top-level package will be optional. Each package, except the
top-level package, should be declared in the `Packages.json` file and in the
following format:

```json
{
    "schema": 1,
    "packages": [
        {
            "name": "foo",
            "path": "subpackages/foo"
        },
        {
            "name": "bar",
            "path": "subpackages/bar"
        }
    ]
}
```

_Note: We may add support for maintaing `Packages.json` using SwiftPM tools in
future._

The packages should declare their versions in the git tags using the format:
`<package_name>/<semver>`. However, the top-level package should continue to
declare its versions using the format: `<semver>`.

We will add a parameter `package` to dependency declaration APIs of
`Package.swift` to support depending on packages inside a multi-package
repository. This parameter should not be used to depend on the top-level
package. Packages within a multi-package repository should also use the same
APIs to refer to each other. Example:

```swift
let package = Package(
    name: "Foo",
    dependencies: [
        // Declares dependency on package Bar.
        .dependency(url: "https://my-multi-package-repo", from: "3.0.0", package: "Bar"),

        // Declares dependency on package Baz.
        .dependency(url: "https://my-multi-package-repo", .exact("3.0.0"), package: "Baz"),

        // Declares dependency on the top-level package.
        .dependency(url: "https://my-multi-package-repo", .exact("3.0.0")),
    ]
)
```
_Note: These APIs will only be available if the tools version of the manifest is
greater than or equal to the version this proposal is implemented in._

SwiftPM tools, when executed at root of a multi-package repository, will behave
as if each package is in the edited state. This means, only the dependencies
that are outside the current repository will be cloned. If packages inside the
multi-package repository depend on each other, they will use the current
checkout instead of the version specified in the manifest. We will use a file
called `Packages.resolved` to record this dependency resolution result.

We will provide an option `--disable-multi-package-edit` to disable the above
described edited behaviour. This is useful to check if the packages work with
the dependencies declared in their manifest files. When this option is used, we
will record the dependency resolution result of each package in their respective
`Package.resolved` file.

Editing a package that is contained in a multi-package repository will put the
entire multi-package repository in edited mode. We can add convenience
symlinks in the `Packages/` directory so the actual edited package is easily
reachable.

## Impact on existing code

None, this was not previously possible.

## Alternatives considered

We considered an alternative where each package in a multi-package repository
will have the same versions. However, that is incorrect use of semver and will
force package authors to bump major version whenever an API is changed in any of
the package. It will also cause issues for package consumers in declaring the
right range for dependencies that are inside a multi-package repository.
