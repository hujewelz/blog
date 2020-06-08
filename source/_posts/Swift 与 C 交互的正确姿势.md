---
title: Swift 与 C 混编的正确姿势
date: 2020-05-08 12:04:03
tags: Swift, SPM
categories: Swift
---

自从笔者第一次尝试 Swift 到现在已经过去 5 年多了，从Swift 的第一个版本到现在的 Swift 5.2，Swift 语言发生了天翻地覆的变化。 Swift 生态也已经很完善，日常开发中用到的各种库基本都支持了 Swift。那些现在还在纠结要不要使用 Swift 的同学可以看看[这篇文章](https://mp.weixin.qq.com/s/N6ToEkN9c-2_rIvkv4o9hA) ，文章中提到的几个问题几乎涵盖了 OC 与 Swift 混编时会遇到的一些问题，文章中都给出了相应的解决方案。

Swift 和 Objective-C 以及 C、C++（Swift 不能直接调用 C++，必须通过 OC进行调用） 混编的阻力非常小。它可以自动桥接 objective-C 的类型，甚至可以桥接很多 C 的类型。这就可以让我们在原有库的基础上，使用 Swift 开发出简洁易用的 API。Swift 和 Objective-C 混编的文章不少，在这篇文章中，我们将学习如何让 C 与 Swift 进行交互。

<!-- more -->

## Bridging Header

当我们在一个 Swift 项目中添加 C 源文件时，Xcode 会询问是否添加 Objective-C 桥接头文件，这跟我们在 Swift 项目中添加 OC 文件一样。接着我们只需要在 Bridging Header 中添加需要暴露给 Swift 代码的头文件：

```c
#include "test.h"
```

在 `test.h` 中声明了一个 `hello` 函数：

```c

#ifndef test_h
#define test_h

#include <stdio.h>

void hello(void);

#endif /* test_h */
```

然后在 `tesh.c` 中实现了它：

```c
#include "test.h"

void hello() {
    printf("Hello World");
}

```

现在我们就可以在 Swift 代码中调用 `hello()` 了。

## Swift Package Manager

上面使用 Bridging header 的方式主要适用于 C 源代码跟 Swift 代码处于同一个 App target 下，对于那些独立的 Swift Framework 就不适用了，在这种情况下就需要使用 Swift 包管理器（[Swift Package Manager](https://swift.org/package-manager/) , 下文简称SPM）了。从 Swift 3.0 开始我们就可以使用 SPM 来构建 C 语言的目标 （target）了。

下面我们将用 Swift 封装一个易用的 OpenGL 程序库。通过这个例子，我们基本上可以掌握如何在一个 Swif 库中与 C 进行交互了。

### 设置 SPM

为导入 C 程序库设置一个 Swift 包管理器项目并不是什么难事，不过还是有不少的步骤要完成。

现在让我们开始创建一个新的 SPM 项目吧。切换要保存代码的目录，执行下面的命令创建一个 SPM 包：

```shell
$ mkdir OpenGLApp
$ cd OpenGLApp
$ swift package init --type library
```
我们通过 `swift package init --type library` 命令创建了一个名为 OpenGLApp 的 Swift 库。我们可以打开 Package.swift 文件看看里面的内容（删除了无关内容）：

```swift
// swift-tools-version:5.2
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "OpenGLAPP",
  	products: [
        .library(name: "OpenGLApp", targets: ["OpenGLApp"])
    ],
    dependencies: [],
    targets: [
        .target(
            name: "OpenGLApp",
            dependencies: [],
    ]
)
```

为了完成一个可以运行的 OpenGL 程序，我们需要依赖 **[GLFW](https://www.glfw.org/)** 和 **[GLEW](http://glew.sourceforge.net/)** 这两个 C 语言库。GLFW 给我们提供了一个窗口和上下文用来渲染，这样我们就不用去书写操作系统相关代码了。GLEW 提供了用于确定其 OpenGL 扩展支持在目标平台上高效的运行时间的机制。 

### 将 C 程序库导出为模块

由于 GLFW 和 GLEW 都是由 C 编写的库，所以我们先要解决如何让 Swift 找到这些 C 语言库，这样，才能在 Swift 调用它们。在 C 里，可以通过  `#include` 一个或多个库的头文件的方式来访问它们。但是 Swift 无法直接处理 C 的头文件，它依赖的是**模块 (Module)**。为了让一个用 C 和 Objective-C 编写的库对 Swift 编译器可见，它们必须安照 [Clang Module](https://clang.llvm.org/docs/Modules.html) 的格式提供一份模块地图 (Module map)。它的主要作用就是列出构成模块的头文件。

因为 GLFW 和 GLEW 并没有提供模块地图，所以我们需要在 SPM 里定义一个专门生成模块地图的目标。它的作用就是把以上的 C 语言库封装成模块，这样就可以在另一个 Swift 模块中调用它们了。

首先，我们需要安装 glew 和 glfw，如果是 macOS 系统可以使用 [homebrew](https://brew.sh/) 来安装。其他的系统就使用相关的包管理器安装就可以了。

接着打开 Package.swift， 在 `targets` 中增加如下内容：

```swift
 ...
targets: [
    ....
    .systemLibrary(
        name: "Cglew",
        pkgConfig: "glew",
        providers: [
            .brew(["glew"])
        ]),
    .systemLibrary(
        name: "Cglfw",
        pkgConfig: "glfw3",
        providers: [
            .brew(["glfw"])
        ]),
]
```

在上面的 Package.swift 中，我们新添加了两个系统程序库目标（system library target）。所谓的系统程序库目标是指那些由系统级别的包管理器安装的程序库，例如我们使用 homebrew 安装的一些程序库。*Sample* 目标是最终的可执行程序，*OpenGLApp* 是我们将要使用 Swift 封装的 OpenGL 库，*Cglew* 和 *Cglfw* 两个系统程序目标就是我们制作的可以在 Swift 中调用的模块。 

在系统程序库目标中 `pkConfig` 和 `providers` 两个参数需要说明一下：

- *providers* 指令是可选的，在目标库没有被安装时，它为 SPM 提供了用于安装库的方式的提示。 
- *pkConfig* 指定了[pk g-config](https://zh.wikipedia.org/wiki/Pkg-config) 文件的名称，Swift 包管理器可以通过它找到要导入的库的头文件和库搜索路径。pkConfig 的名称我们可以在库的安装路径的 `lib/pkconfig/xxx.pc` 中找到，以我电脑中安装的 glew 为例，它的位置是 `/usr/local/Cellar/glew/2.1.0/lib/pkgconfig/glew.pc`，所以上面 pkConfig 中设置的就是 `glew`。

接下来我们需要在 Sources 目录下为系统程序库目标创建一个保存文件的目录，该目录名称必须跟上面 Package.swift 中定义的目标的 `name` 属性一致。这里我以 Cglfw  为例：

```shell
$ cd Sources && mkdir Cglfw
```

在 Cglfw 目录中添加一个 `glfw.h` 文件，并添加如下内容：

```c
#include <GLFW/glfw3.h>
```

接着添加一个 `module.modulemap` 文件，它应该是下面的样子：

```
module Cglfw [system] {
    header "glfw.h"
    export *
}
```

我们添加 glfw.h （名称可以自己定义）文件的目的是绕过模块地图中必须包含绝对路径的限制，否则的话，我们就必须在 modulemap 文件中的 `header` 中指定 glfw3.h 头文件的绝对路径，在我的电脑上就是 `/usr/local/Cellar/glfw/3.3.2/include/GLFW/glfw3.h`，这样就将 GLFW 的路径硬编码到模块地图中了。使用了我们添加的 glfw.h 文件，SPM 就会从 pkg-config 文件中读取正确的头文件搜索路径，并将它添加到编译器的调用中。

我们可以按照同样的方式将 GLEW 导出为模块，这里我就不演示了。上面是将安装在系统中的 C 程序库导出为模块，不过有些情况下我们只有 C 程序库的源代码，这个时候我们仍然可以使用 SPM 将 C 程序源码导出为模块。

**C 源码导出为模块**

将 C 源代码导出为模块也非常简单，其实也是编写模块地图的过程，不过这个过程我们可以借助 SPM 自动帮我们完成。

我们可以从[这里](https://sourceforge.net/projects/glew/)下载 GLEW 的源码。跟上面的步骤一样，在 Sources 目录下创建一个 Cglew 子目录，并将解压后的 GLEW 源代码中 include 和 src 目录拷贝到 Cglew 目录下。然后我们在 Package.swift 中添加如下内容：

```swif
.target(name: "Cglew")
```

在上面的过程中我们并没有编写模块地图，并不是说通过这种方式不需要模块地图，而是 SPM 自动帮我们完成的。我们将需要暴露给外部的头文件放到 include 目录下，编译时 SPM 就会自动生成模块地图。当然我们也可以通过 `publicHeadersPath` 参数来指定需要暴露给外部头文件的路径。

接着我们可以来完成 OpenGLApp 这个目标了。在 OpenGLApp 目录中添加一个 `GLApp.swift` 文件。现在，我们就可以在 Swift 文件中使用 `import Cglew` , `import Cglfw`，并调用 GLFW 和 GLEW 中提供的 API 了。有一点不要忘记，我们需要在 Package.swift 文件中 OpenGLApp 这个目标的 `dependencies` 添加我们都依赖：

```swift
.target(
    name: "OpenGLApp",
    dependencies: ["Cglfw", "Cglew"],
    linkerSettings: [
        .linkedFramework("OpenGL")
    ]),
```
> 为了方便在 Xcode 中编写并调试程序，可以使用 `swift package generate-xcodeproj` 命令来生成一个 Xcode 工程。

> 在通过 `import Cglew` 引入 Cglew 模块并构建项目，你会发现 Xcode 报了大量错，这个时候可以在  Cglew 目标中的 `glew.h` 文件最上面添加 `#define GLEW_NO_GLU`。

后面的主要工作就是编写 OpenGL 代码了，这里就不展开了，毕竟不是本文的重点。

接着我们可以添加一个用于运行该库的可执行程序的目标。我们在 Sources 目录下添加 Sample 子目录，并添加一个 `main.siwft` 文件，并在 Package.swift 中的 `targets` 添加一个 Sample 目标：

```swift
.target(
	name: "Sample",
  dependencies: ["OpenGLApp"]),
```

我在 `main.siwft` 中调用了自己封装的 OpenGLApp 的 Swift 库：

```swift
import OpenGLApp

let app = GLApp(title: "Hello, OpenGL", width: 600, height: 600)
app.run()
```

> SPM 会将包含有 main.swift 文件的目标作为可执行文件目标。所以我们在用 SPM 开发库时，库文件中不要有 main.swift 文件，否则的话，SPM 会将该目标作为可执行文件而不是一个库，这样就无法正确地和其他库或可执行文件进行链接了。

如果我们继续在终端中执行 `swift run` 命令，这时 SPM 就会构建并执行这个应用程序。

![Hello,window](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/swiftc/window.png)

下面是完整的 Package.swift：

```swift
// swift-tools-version:5.2
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "GLAPP",
    products: [
        .library(name: "OpenGLApp", targets: ["OpenGLApp"])
    ],
    dependencies: [],
    targets: [
        .target(
            name: "Sample",
            dependencies: ["OpenGLApp"]),
        .target(
            name: "OpenGLApp",
            dependencies: ["Cglew", "Cglfw"],
            linkerSettings: [
                .linkedFramework("OpenGL")
            ]),
        .systemLibrary(
            name: "Cglew",
            pkgConfig: "glew",
            providers: [
                .brew(["glew"])
            ]),
        .systemLibrary(
            name: "Cglfw",
            pkgConfig: "glfw3",
            providers: [
                .brew(["glfw"])
            ]),
    ]
)
```

总结一下，要想让 Swift 模块能调用 C 程序，只需要将 C 程序代码导出为模块即可。而导出模块只需要按照[Clang Module](https://clang.llvm.org/docs/Modules.html) 的格式提供一份模块地图。

## 回顾

在 Swift 代码中使用 C 程序代码其实是一件很简单的事情，比起用 Swift 重写一个已经存在的 C 程序库，为什么不直接在 Swift 中使用它们呢。当然在实际使用过程中也肯定会遇到一些问题，比如 C 中的指针，回调函数等等，不过这些并不是什么大的问题，不知道如何使用只是表明我们对 Swift 某些地方还不熟悉。



