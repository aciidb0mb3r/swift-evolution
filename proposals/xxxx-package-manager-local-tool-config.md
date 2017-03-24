# Package Manager Local Tool Configuration

* Proposal: [SE-XXXX](XXXX-package-manager-local-tool-configuration.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review manager: TDB
* Status: Draft

## Introduction

Swift package manager has three command line tools: `swift-package`, `swift-build`, `swift-test`. This proposal aims to add support for storing configuration data for these tools.

## Motivation

A command line tool usually uses a file to store the configuration data of that tool, for e.g. git has `.gitconfig`. It is natural for Swift package manager to provide this ability for its own tools.

Another motivation is having a temporary workaround to store, document and apply any extra the command line flags needed by a package i.e. the flags specified using the options `-Xcc`, `-Xswiftc` and `-Xlinker`. **Please note that this is only a temporary workaround until we have a proper way of specifying the information in the manifest file.**  
These flags are usually only required by complex packages where package manager's pkg-config approach doesn't work. 
Currently these packages document these flags in their readme. Tool configuration would be a good place to start including this information with the package.  
It is also worth mentioning that this information is not allowed to be stored in manifest because such custom flags can introduce **very** unexpected behaviors for downstream dependencies. It becomes very hard for package manager to reason about these flags and do the right thing.

## Proposed solution

We propose to introduce a config file, `.swiftpm-config`, which will be read by the package manager if placed along side the manifest file. This file will be an *optional* JSON file which can be checked-in into SCM.

We propose to introduce a local config file `.swiftpm-config.local`, which serves same functions as `.swiftpm-config`. However if present, package manager will ignore `.swiftpm-config` and read this file instead for getting the configuration data. This can be used to always ignore config provided by the package and use local config instead. It is advisable to never check this in and add it to git ignore file.

## Detailed design

1. In presence of a config file i.e. `.swiftpm-config` and `.swiftpm-config.local`, the package manager will always apply the configuration stored in them, in respective priority.

2. We would never inherit this data across a package dependency.

3. We will add an option to ignore config files:
    
        $ swift build --no-config
        $ swift test --no-config

4. We will initially add configuration for these three options: `-Xcc`, `-Xswiftc` and `-Xlinker`, with following format:

```json
{
    "custom_flags": {
        "-Xcc": [
            "-DDEBUG",
            "-I",
            "/path/to/include"
        ],
        "-Xswiftc": [
            "-DENABLE_THAT_THING",
        ],
        "-Xlinker": [
            "-L/path/to/libs",
            "-lFoobar"
        ]
    }
}
```

## Future Directions

Following are the future direction we would like to implement after landing the initial infrastructure:

* We can add an option to provide path to a config file at a path, for e.g.:
    
        $ swift build --package-config /path/to/config

* We can provide a global config file (`~/.swiftpm-config`).

* We would eventually add `git config` like options under `swift package config`.

## Impact on existing code

None.

## Alternatives considered

None.
