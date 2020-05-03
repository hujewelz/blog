---
title: Swift Package Manager
date: 2019-08-24 10:57:03
tags:
categories: Swift
---

Swift Package Manager 是用于管理 Swift 代码分发的工具。它与 Swift 构建系统集成在一起，可以自动执行依赖项的下载，编译和链接过程。

<!-- more -->

从 Swift 3.0 开始就可以使用 Swift Package Manager 了。

## 几个概念

在开始创建和使用 Swift Package 之前，有必要先了解几个跟 Swfit Package 相关的基础概念。

### Modules

Swift 将代码组织到模块中。每个模块都指定一个命名空间，并通过权限修饰符来定义可以在模块外部使用该代码的哪些部分。

程序可以将所有代码都放在一个模块中，也可以将其他模块作为依赖项导入。除了少数系统提供的模块外，例如 macOS 上的 Darwin 或 Linux 上的 Glibc，大多数依赖都需要下载并构建代码才能使用。

当我们为解决特定问题的代码使用单独的模块时，该代码可以在其他情况下重用。 使用模块，我们可以在其他开发人员的代码上构建代码，而不必重复造轮子。

### Packages

一个 Package 由 Swift 源文件和 Package.swift 文件组成。它使用 `PackageDescription` 模块定义了包的名称及其内容。一包中有一个或多个 target。每个 target 都指定一个 Product，并可以声明一个或多个依赖项。

### Products

一个 target 可以构建为库或可执行文件，即 product。一个库包含一个可以被其他 Swift 代码导入的模块。可执行文件是可以由操作系统运行的程序。

### Dependencies

一个 target 的依赖是 Package 中的代码所需的模块。依赖是包括相对于 Package 源的相对 URL 或绝对 URL，以及对可以使用的版本。
包管理器的作用是通过自动化下载和构建项目的所有依赖项的过程来减少成本。 这是一个递归过程：依赖项可以具有自己的依赖项，每个依赖项也可以具有依赖项，从而形成依赖关系图。 包管理器下载并构建满足整个依赖关系图所需的所有内容。

## 创建 Package

我们可以通过终端命令或者 Xcode 来创建 Swift Package。在创建之前，请确保在你已经从 [Swift.org](https://swift.org/getting-started/) 完成 Swift toolchain 工具集的安装。

### 使用命令行

在完成 Swift toolchain 工具集安装后就可以开始创建 Package 了。请新建一个文件夹用于保存项目文件：

```shell
$ mkdir Hello
$ cd Hello
$ swift package init --type=library
```

上面我们通过 `swift package init` 命令创建了一个名为 Hello 的 Swift 库。这里 `type` 我们设置的是 `library` ，所以该 Package 是作为一个模块提供给其他 Swift 代码使用的。

我们可以在终端执行 `swift package init --help` 来查看更多内容：

```shell
$ swift package init --help

OVERVIEW: Initialize a new package

OPTIONS:
  --name   Provide custom package name
  --type   empty|library|executable|system-module|manifest

```

这里 `type` 选项有 5 种值可供选择，我们比较常用的是 `library` 和 `executable` 。
`executable` 表明生成的是可执行文件，它是可以由操作系统运行的程序。
`manifest` 可以快速生成一个 `Package.swift` 文件。

### 使用 Xcode

新版的 Xcode 已经集成了 Swift Package Manager，使用 Xcode 我们能很方便的创建 Package 和添加依赖。

通过选择 **File▸New▸Swift Package...** 创建一个 Swift Package。

{%img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/1.jpg 600 %}

这里我将新创建的 Package 添加到 SPDemo 项目中，点击 **Create** 后，Xcode 就会生成给我们生成一个新的 Package。

{%img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/2.jpg 300 %}

## 发布

当我们的 Package 完成开发和测试后就可以进行发布了，这样其他人就可以通过添加依赖使用你代码了。

发布 Package 非常简单，只需要借助 Git 工具。我们需要先在 [Github](https://www.github.com) 创建好仓库。打好 tag 后，将 tag 推送到 github 仓库中。
这里的 tag 代表了 Package 的版本，你可以在 [Semver](https://semver.org/) 上了解更多关于版本号的设置。

## 添加 Package 依赖

在现有项目中添加 Package 依赖非常简单。

在 Xcode 中选择 **File▸Swift Packages▸Add Package Dependency...**， 在弹框中输入我们需要添加的 Packages 的 github 仓库地址，点击 **Next** ，弹框中 Rules 指定了获取特定版本号、分支或 git 提交的源代码。

{%img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/4.jpg 600 %}

后面 Xcode 就根据上面规则中下载的源代码来进行构建 Package。

继续点击 **Next** 后，弹框中列出了该 Package 下的所有 Products, 我这里以 [RxSwift](https://github.com/ReactiveX/RxSwift) 为例，勾选需要依赖的 Product 后点击 **Finish**。

{% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/5.jpg 600 %}

Xcode 会自动下载源代码，并执行构建。在 **TARGETS▸General▸Frameworks, Libraries, and Embedded Content** 中可以看到我们添加的 Swift 库。在上面添加 RxSwift 时，我选择了 `RxCocoa` 和 `RxRelay` ，所以 Xcode 也只会自动给我们添加这两个库，如果需要引入 `RxSwift` 则需要我们点击下面的 **+** 手动添加。

{% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/6.jpg 600 %}

上面是在 Xcode 工程中添加 Swift Packages 的过程。如果是非 Xcode 工程，该这么添加依赖呢？这就需要详细了解 Package.swift 文件了。

## Package.swift

`Package.swift` 使用 `PackageDescription` 模块定义了包的名称及其内容，是一个 Swift Package 中必不可少的文件。

一个简单的 Package.swift 文件都有下面的内容：

```swift
let package = Package(
    name: "Hello",
    products: [
        // Products define the executables and libraries produced by a package, and make them visible to other packages.
        .library(
            name: "Hello",
            targets: ["Hello"]),
    ],
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        // .package(url: /* package url */, from: "1.0.0"),
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages which this package depends on.
        .target(
            name: "Hello",
            dependencies: []),
        .testTarget(
            name: "HelloTests",
            dependencies: ["Hello"]),
    ]
)
```

- **name** 定义了 Package 的名称；
- **products** 定义 Package 产生的可执行文件和库，并使它们对其他软件包可见;
  以 RxSwift 为例，其 Package.swift 中定义了如下 Products：

  ```swift
  .library(name: "RxSwift", targets: ["RxSwift"]),
  .library(name: "RxCocoa", targets: ["RxCocoa"]),
  .library(name: "RxRelay", targets: ["RxRelay"]),
  .library(name: "RxBlocking", targets: ["RxBlocking"]),
  .library(name: "RxTest", targets: ["RxTest"]),
  ```

  所以在上面添加 RxSwift 依赖时才能列出多个 Product：

  {% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/spm/5.jpg 400 %}

- **dependencies** 定义了该 Package 所依赖的其他 Packages;
  对于 Xcode 项目，可以直接在 Xcode 中选择通过 **File▸Swift Packages▸Add Package Dependency...** 添加，对于其他非 Xcode 项目可以通过该属性来指定。

  我们可以使用 `.package(name:, url:, from:)` 来添加远程依赖，Swift Package Manager 在构建时会从指定的 url 中下载代码进行构建；也可以使用 `.package(path:)` 从本地添加依赖，这里的 `path` 就是本机上项目中 `Package.swift` 的路径。

- **targets** 是 Package 的基础编译模块。 可以定义模块或测试套件。一个 target 可以依赖 package 中的其他 targets，这是通 过 `dependencies` 属性来指定的。

  以 RxSwift 为例， `RxRelay` 依赖了 `RxSwift` 这个 target。

  ```swift
  .target(name: "RxRelay", dependencies: ["RxSwift"]),
  ```

  当然也可以依赖我们在上面 **dependencies** 中指定的第三方 Packages，就像下面的例子：

  ```swift
  import PackageDescription

  let package = Package(
      name: "MyAwesomeProject",
      dependencies: [
        .package(name: "PerfectHTTPServer", url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git", from: "3.0.0")
      ],
      targets: [
          .target(
              name: "MyAwesomeProject",
              dependencies: ["PerfectHTTPServer"]),
      ]
  )
  ```

  这里 `MyAwesomeProject` 的 target 中就依赖了 **dependencies** 中名为 `PerfectHTTPServer` 的 Product。

## 创建多个 Targets

上面我们已经了解了从创建到添加 Swift Package 的整个过程，也对一个 Swift Package 中 `Package.swift` 有了一定的了解。那么我们怎样才能创建一个包含多个 Target 的 Package 呢？

一个简单的 Package 的文件结构如下：

```
Hello
├── Sources
│   └── Hello
│       └── Hello.swift
└── Package.swift

```

Sources 目录下的 Hello 就是一个 target, 如果我们像增加其他的 target 只需要在 Sources 目录下添加一个新的目录即可，例如:

```
Hello
├── Sources
│   │── Hello
│   │    └── Hello.swift
│   │── Other
└── Package.swift

```

然后在 Package.swift 中增加一个 target：

```swift
.target(name: "Other",dependencies: [])
```

这里 target 的名称必须和 `Sources/<target-name>` 目录名称一致。

## 写在最后

Swift Package Manager 确实方便了我们在项目中安装和管理依赖，相比与 [Cocoapods](https://cocoapods.org/)，它不会改变我们的项目的结构，与 [Carthage](https://github.com/Carthage/Carthage) 一样的轻巧，但它又比后者操作更简单，免去了后者那烦人的手动设置 framework 路径的操作。毕竟它是 Apple 的亲儿子，在体验上不知道其他第三方工具好了多少。笔者还是很推荐使用 Swift Package Manager 来管理你的项目依赖的，我们平时常用的第三方库也都支持 Swift Package Manager。
