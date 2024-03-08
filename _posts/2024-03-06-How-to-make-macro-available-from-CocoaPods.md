---
title: How to make Swift Macro available using CocoaPods 
author: Klein
date: 2024-03-06 21:00:00 +0800
categories: [Tech, Swift]
tags: [Tech, Swift, Rust]
summary: Provide macros to other project's or development pods using CocoaPods instead of using SwiftPM.
---

By the release of Swift 5.9, it provides the feature `Swift Macro`, which is really useful for the developer to reduce boilerplate code and helping to improve the readability of the code.

However, as we know, currently most long running projects are using CocoaPods as their dependency manager, while the Swift Macro support officially relies on SwiftPM. This prevents macros from being directly used in the project code and development pods.

So this article is about to introduce how to make Swift Macro available using CocoaPods, for host project and other pods.

## Key Point - using an executable macro plugin

Inspired by the information in [this discussion](https://forums.swift.org/t/how-to-import-macros-using-methods-other-than-swiftpm/66645/10/) and [this post](https://www.polpiella.dev/binary-swift-macros), it shows that we can provide an executable binary to the Swift Compiler in Xcode settings: add `-load-plugin-executable <path-to-plugin-executable>#<executable-module-name>` to `OTHER_SWIFT_FLAGS`.

For example:

```ruby
'OTHER_SWIFT_FLAGS' => '-load-plugin-executable Resources/Macros/MyMacroPlugin#MyMacroPlugin',
```

That means we can build a plugin executable and provide it through CocoaPods, update the settings in Pods project and host project before or after the `pod install` command.

Okay, let's do it.

(All the example code can be found in [this repo](https://github.com/Mioke/SwiftyArchitectureMacros))

## Create a macro plugin executable

We can easily create a demo macro project using Xcode 15 or command line `swift package init --type macro`. And then update the `Package.swift` like this:

```swift
// swift-tools-version: 5.9
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription
import CompilerPluginSupport

let package = Package(
    name: "SwiftyArchitectureMacros",
    platforms: [.macOS(.v10_15), .iOS(.v13), .tvOS(.v13), .watchOS(.v6), .macCatalyst(.v13)],
    products: [
        // Product which is a executable plugin for other project's compiler to integrate.
        .executable(
            name: "SwiftyArchitectureMacros",
            targets: ["SwiftyArchitectureMacros"]),
    ],
    dependencies: [
        // Depend on the Swift 5.9 release of SwiftSyntax
        .package(url: "https://github.com/apple/swift-syntax.git", from: "509.0.0"),
    ],
    targets: [
         .executableTarget(
             name: "SwiftyArchitectureMacros",
             dependencies: [
                 .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                 .product(name: "SwiftCompilerPlugin", package: "swift-syntax")
             ]
         ),
        // A test target used to develop the macro implementation.
        .testTarget(
            name: "MacrosTests",
            dependencies: [
                "SwiftyArchitectureMacros",
                .product(name: "SwiftSyntaxMacrosTestSupport", package: "swift-syntax"),
            ]
        ),
    ]
)
```

There are some important informations:

1. The target `SwiftyArchitectureMacros` must be a executable target, normally it's a `.macro` target. And the product of the executable should target on the executable `SwiftyArchitectureMacros`.

2. Second, by several tests, the executable target should contain original macro files, not macro definition files. So it means that the definition of the macro should be contained in the pod we build.

By using this `swift build -c release` command, we can get a `SwiftyArchitectureMacros` executable file in `.build/release/`.

## Create a pod to host the macro plugin executable

### Using a prepared macro executable

Next, we will create a podspec file to host the executable file and add some configurations.

We can create a `.podspec` now, and in my case it is `SwiftyArchitectureMacrosPackage.podspec`. The key content is below.

```ruby
s.source_files = 'Sources/MacrosDefine/*'
s.preserve_paths = 'Products/**/*'

xcode_config = {
  'OTHER_SWIFT_FLAGS' => <<-FLAGS.squish
  -Xfrontend -load-plugin-executable
  -Xfrontend $(PODS_ROOT)/SwiftyArchitectureMacrosPackage/Products/SwiftyArchitectureMacros#SwiftyArchitectureMacros
  FLAGS
}

s.user_target_xcconfig = xcode_config # <-- add to the `Host project`.
s.pod_target_xcconfig = xcode_config
```

Key points:

1. The `source_files` should contain the macro definition files, which are the files that contains the macro definitions like:

   ```swift
   @freestanding(expression)
   public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "SwiftyArchitectureMacros", type: "StringifyMacro")
   ```

2. The `preserve_paths` should contain the executable file, which we build before and move it to a folder, like `Products/`.
3. The `user_target_xcconfig` and `pod_target_xcconfig` should contain the same configurations, which integrate the executable to compiler plugin.
4. The `s.user_target_xcconfig` is used to modify settings of the host project, while the `s.pod_target_xcconfig` is used to modify settings of the current pod target.

### Using a script to build a executable

Inspired by [this post](https://soumyamahunt.medium.com/support-swift-macros-with-cocoapods-3911f9317042), we can also use a script to build the executable.

We can create another `.podspec` file, which is `SwiftyArchitectureMacros.podspec` in my case, and the key content is below.

```ruby
s.source_files = 'Sources/MacrosDefine/*'
s.preserve_paths = 'Package.swift', 'Sources/**/*', 'Tests/**/*'

product_folder = "${PODS_BUILD_DIR}/Products/SwiftyArchitectureMacros"

script = <<-SCRIPT.squish
env -i PATH="$PATH" "$SHELL" -l -c
"swift build -c release --product SwiftyArchitectureMacros
--package-path \\"$PODS_TARGET_SRCROOT\\"
--scratch-path \\"#{product_folder}\\""
SCRIPT

s.script_phase = {
 :name => 'Build SwiftyArchitectureMacros macro plugin',
 :script => script,
 :input_files => Dir.glob("{Package.swift, Sources/**/*}").map {
   |path| "$(PODS_TARGET_SRCROOT)/#{path}"
 },
 :output_files => ["#{product_folder}/release/SwiftyArchitectureMacros"],
 :execution_position => :before_compile
}
```

Besides the key points introduced above, this section also needs attention to several other key points:

1. The `preserve_paths` should contain the `Package.swift` and files that are used to build the executable.
2. And don't forget to update the build config path to `#{product_folder}/release/SwiftyArchitectureMacros#SwiftyArchitectureMacros`.

The benefit is that we don't need to prepare the executable file, it will be built by the script when the main project starts building, and it won't have any compatible issue. However you should know that the script will be executed every time when the project builds, and it may takes a long time when there's no build cache. So I suggest as a SDK provider, you should provide both options.

## Integrate to other targets

### Host project

If your main codes are in the host project, you can integrate the macro plugin to the host project by adding the following code to the `Podfile`.

```ruby
pod 'SwiftyArchitectureMacrosPackage'
#or
pod 'SwiftyArchitectureMacros'
```

Because of the `OTHER_SWIFT_FLAG` setting are already inserted by the `podspec` file into the host project settings, you don't need to do anything else.

```swift
import SwiftyArchitectureMacrosPackage

func test() {
  let a = 1
  let b = 2
  let desc = #stringify(a + b)
  print(desc)
}
```

It works fine~

### Used by other pods or development pods

First, add the dependency in the other's podspec:

```ruby
s.dependency 'SwiftyArchitectureMacrosPackage'
```

And then it will be a little tricky, because we can't directly insert the `OTHER_SWIFT_FLAG` into other pod target settings because:

1. In another's podspec, hard code the executable path is not a good idea, because the path may changes when you switching the macro pod between local and remote.
2. If a lot of pods are depending on the macro pod, when macro pod setting changes you must update all the pods' podspec file which is a big trouble.

So we need to do some tricks. We can use `pod install`'s `post_install` hook to do this. Add these code to your `Podfile`, it aims to add the `OTHER_SWIFT_FLAG` setting to the `Pods.xcodeproj`, and all the pod targets will inherit from it.

```ruby
post_install do |installer_representation|

  macro_product_folder = "${PODS_BUILD_DIR}/Products/SwiftyArchitectureMacros"

  installer_representation.pods_project.build_configurations.each do |config|
    config.build_settings['OTHER_SWIFT_FLAGS'] = "$(inherited) -load-plugin-executable #{macro_product_folder}/release/SwiftyArchitectureMacros#SwiftyArchitectureMacros"
  end

end
```

Now try `pod install` and see if all the pod targets are inheriting correctly from the `Pods.xcodeproj`. If so, you can use the macros in your codes now.
