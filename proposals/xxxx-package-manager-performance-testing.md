# Package Manager Performance Testing Conventions

* Proposal: [SE-XXXX](xxx-package-manager-performance-testing.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: Discussion

## Introduction

This is a proposal for adding package manager support for building and running performance tests.

## Motivation

The package manager supports building and running tests using the `swift test` subtool. The sources are built in debug mode with swift compiler's
`-enable-testing` flag allowing testable imports in test modules (which gives it internal level access to the imported module). 
This works well for regular correctness/unit tests, however for performance tests, we are usually interested in release builds, which take advantage of compiler 
optimizations.

We can provide an option to compile and run the tests in release mode with `-enable-testing` flag. But that disables some of the compiler optimizations, which 
may be of interest to packages having very performance sensitive code and which critically depends on those optimization.

There is also a requirement to have a way to run performance tests separate from correctness tests, because:
* Performance tests usually take more time to run than correctness tests.
* We can't determine which tests do and do not have performance tests.

As a concrete example, package manager's own performance tests are disabled from building or running on Swift CI.

## Proposed solution

### Performance Test Modules

We propose to extend the test modules conventions and consider a directory as "performance test module" which has a suffix "PerformanceTests".
This means we now have two types of tests: correctness and performance tests — each test module belongs to one of these categories.

The correctness tests will run via the existing `swift test` with optional `--correctness` flag:

```
$ swift test
or
$ swift test --correctness
```

The performance tests will run using the `--performance` flag:

```
$ swift test --performance
```

To run all the tests (correctness + preformance), pass both the flags:

```
$ swift test --correctness --performance
```

Also, configuration can be used to run tests in release mode:

```
$ swift test --performance -c release
```

---
### Per Module Testability Setting

_Note: This setting will only be considered when building in release configuration._

Consider the two types of testing methods:  
* `Black box` tests, which should test only public API, don't require `testability` enabled.
* `White box` tests, which are allowed to use internal API, might require `testability`.

For performance tests, we would like to support a mode in which black box tests run in the most optimal configuration i.e. with testability disabled.

To handle these cases, we propose to add a new property called `requiresTestability` to class `PackageDescription.Target`. The property would be an enum nested in `Target`:

```swift
/// Defines the testability setting for a target.
enum Testability {
    /// Let the package manager determine if testability should be enabled or disabled.
    case auto

    /// Enable testability.
    case true

    /// Disable testability.
    case false
}
```

Enabling this setting would concretely mean that, this target uses testable imports and must have testability enabled in order to be able to build. By default this property will be set to `.auto` for all targets.

```swift
public final class Target {
    ...

    /// The testability setting of this target.
    public let requiresTestability: Testability = .auto
}
```

The behavior for `auto` setting is defined on testability as following:

* Disabled for all regular source modules.
* Disabled for all performance test modules.
* Enabled for all correctness test modules.

This means we might need to build some modules twice, for e.g. when a source module has both correctness and performance tests.

It would be invalid for a module that has `requiresTestability` disabled, to depend on a module which has `requiresTestability` enabled. For e.g. this would be an invalid manifest:

```swift
let package = Package(
    name: "MyPackage",
    targets: [
        Target(
            name: "Foo"),
        Target(
            name: "TestSupport",
            dependencies: ["Foo"],
            requiresTestability: .true),
        Target(
            name: "FooPerformanceTests",
            dependencies: ["Foo", "TestSupport"]
            requiresTestability: .false), // <--- This is invalid.
    ]
)
```
In this case `requiresTestability` for `FooPerformanceTests` should be set to `.true` because it depends on `TestSupport` which requires testability.

## Detailed Design

Currently one XCTest product (bundle on macOS, executable on Linux) is created and run for all the test modules with name `<package-name>PackageTests.xctest`.

We propose to create another test product with name `<package-name>PackagePerformanceTests.xctest` which will contain only performance tests.
This will allow package manager to run these test products separately.

## Impact on existing code

There will be no impact on exisiting code as this is purely additive.

## Alternative considered

1. Do nothing special, just suggest users use XCTest features as normal. SwiftPM will have no ability to slice and dice out performance tests.

2. Build an XCTest mechanism which solves the con of #1 by allowing us to simply disable all performance test blocks.

3. Add a manifest flag for "requiresTestability", but nothing else. Allows users to build their own conventions (using, e.g., Swift language configuration checks).

4. We could have defined a new configuration which means "release + testability" and then the test command just always runs the most tests available to it given the current configuration. This is probably the simplest model, but maybe gives up the ability for us to automatically do the build twice, a feature useful for advanced users.
