---
title: Metal 的执行模型
date: 2020-04-30 12:13:00
tags:
- iOS
- Metal
- 图形学
categories: 图形学
---

在 Metal 的架构中，`MTLDevice` 协议定义了代表单个 GPU 的接口。`MTLDevice` 协议提供了一些方法用于查询设备属性，创建其他特定于设备的对象（例如缓冲区和纹理）以及用于编码和排队渲染和要提交到 GPU 中执行的计算命令。

<!--more-->

命令队列 (*command queue*) 由命令缓冲区 (*command buffers*) 队列组成，命令队列组织这些命令缓冲区的执行顺序。 命令缓冲区包含用于在特定设备上执行的编码命令 (*encoded command*)。命令编码器 (*command encoder*) 将渲染、计算和 blitting 命令附加到命令缓冲区中，这些命令缓冲区最终提交给设备执行。

`MTLCommandQueue` 协议为命令队列定义了接口，主要支持创建命令缓冲区对象的方法。`MTLCommandBuffer` 协议为定义了命令缓冲区的接口，并提供了创建命令编码器、将命令缓冲区排队执行、检查状态和其他操作的方法。`MTLCommandBuffer` 协议支持以下命令编码器类型，这些类型是用于将不同类型的 GPU 工作负载编码到命令缓冲区的接口：

* `MTLRendererCommandEncoder` 协议对单个渲染过程的图形（3D）渲染命令进行编码。
* `MTLComputeCommandEncoder` 协议对数据并行计算工作负荷进行编码。
* `MTLBlitCommandEncoder` 协议对缓冲区和纹理之间的简单复制操作以及诸如 mipmap 生成之类的实用操作进行编码。

在任何时间点，只有单个命令编码器可以被激活，并将命令附加到命令缓冲区中。每个命令编码器必须先结束，然后才能创建另一个用于同一命令缓冲区的命令编码器。当然 “每一个命令缓冲区只有一个激活的命令编码器”的规则也有例外，例如 `MTLParallelRenderCommandEncoder` 协议，这会在[使用多个线程对单个渲染过程进行编码]()的章节中进行讨论。

完成所有编码完成后，就可以提交 `MTLCommandBuffer` 对象了，这会将命令缓冲区标记为准备被 GPU 执行。`MTLCommandQueue` 协议控制已经提交的 `MTLCommandBuffer` 对象中的命令相对于已在命令队列中的其他`MTLCommandBuffer` 对象何时被执行。

下图显示了命令队列、命令缓冲区和命令编码器对象之间的紧密关系。图中顶部的组件的每一列（缓冲区，纹理，采样器，深度和模板状态，管线状态）表示了特定于特定命令编码器的资源和状态。

<img src="https://developer.apple.com/library/archive/documentation/Miscellaneous/Conceptual/MetalProgrammingGuide/Art/Cmd-Model-1_2x.png" style="width: 80%" />

# Device 对象代表了 GPU

一个 `MTLDevice` 对象代表了一个能执行命令的 GPU。`MTLDevice` 协议提供了用于创建命令队列、从内存中分配缓冲区、创建纹理和查询设备功能的方法。调用 `MTLCreateSystemDefaultDevice` 函数可以获取系统上的首选系统设备。

```swift
let device = MTLCreateSystemDefaultDevice()
```


# 临时对象和非临时对象

在 Metal 中一些对象是临时的和极度轻量的。而其他的对象使用起来则代价更高，而且可以持续很长时间，可能是整个应用程序的生命周期。

命令缓存区和命令编码器对象就是临时的，仅供一次使用。它们在分配内存和回收时代价都是非常小的，所以它们的创建方法返回的是自动释放对象。

以下对象不是临时的。在性能敏感的代码中应该重复使用这些对象，并避免重复创建它们：

* 命令队列 (Command queues)
* 数据缓冲区 (Data buffers)
* 纹理 (Textures)
* 采样器状态 (Sampler states)
* 库 (Libraries)
* 计算状态 (Compute states)
* 渲染管线状态 (Render pipeline states)
* 深度/模板状态 (Depth/stencil states)


# 命令队列

命令队列接受 GPU 将执行的命令缓冲区的有序列表。发送到单个队列的所有命令缓冲区都会按照它们入队的顺序来执行。通常，命令队列是线程安全的，允许同时对多个活动命令缓冲区进行编码。

我们可以调用 `MTLDevice` 对象的 `newCommandQueue` 方法或者 `newCommandQueueWithMaxCommandBufferCount` 方法来创建一个命令队列。通常，命令队列的寿命很长，因此不应重复创建和销毁它们。


# 命令缓冲区

命令缓冲区存储已编码的命令，直到缓冲区被提交以供 GPU 执行为止。一个命令缓冲区可以包含许多不同类型的已编码的命令，具体取决于用于构建该命令缓冲区的编码器的数量和类型。在典型的应用程序中，即使整个渲染帧涉及多个渲染过程、计算处理功能或 blit 操作，整个渲染帧也会被编码到单个命令缓冲区中。

命令缓冲区是临时的一次性对象，不支持重用。一旦命令缓冲区被提交执行，唯一有效的操作就是等待命令缓冲区通过同步调用或处理程序块被调度或完成以及检查命令缓冲区的执行状态。

命令缓冲区还表示应用程序唯一可独立跟踪的工作单元，它们定义了 Metal 内存模型建立的一致性边界，详见资源对象：缓冲区和纹理。


## 创建命令缓冲区

我们使用 `MTLCommandQueue` 的 `commandBuffer` 方法来创建 `MTLCommandBuffer` 对象。`MTLCommandBuffer` 对象只能提交到创建它的 `MTLCommandQueue` 对象中。

由 `commandBuffer` 方法创建的命令缓冲区保留了执行所需的数据对于某些情况，在执行 `MTLCommandBuffer` 对象的过程中在如果要在其他位置保留这些对象，应该通过调用 `MTLCommandQueue` 的 `commandBufferWithUnretainedReferences` 方法来创建命令缓冲区。仅对性能非常关键的应用程序使用 `commandBufferWithUnretainedReferences` 方法，该方法可以确保关键对象在应用程序中的其他位置具有引用，直到命令缓冲区执行完成为止。否则，不再具有其他引用的对象可能会被提前释放，这就导致了命令缓冲区执行的结果是未定义的。


## 执行命令

`MTLCommandBuffer` 协议使用以下方法来建立命令队列中命令缓冲区的执行顺序。命令缓冲区在提交之前不会开始执行。一旦提交，命令缓冲区将按其入队的顺序执行。

* `enqueue` 方法在命令队列中为命令缓冲区保留了一个位置，但不提交命令缓冲区以供执行。当这个命令缓冲区最终被提交时，它将在相关命令队列中排在它前面的命令缓冲区之后执行。
* `commit` 方法使命令缓冲区尽可能快地执行，但是仍然必须在同一命令队列中排在它前面的已提交的命令缓冲区之后执行。如果命令缓冲区先前尚未入队，则 `commit` 会进行隐式入队调用。


## 为命令缓冲区的执行注册处理代码块

下面列出的 `MTLCommandBuffer` 的方法用于监视命令执行。调度和完成处理程序会按照执行顺序在未定义的线程上被调用。在这些处理程序中不应该进行耗时的操作；如果需要执行耗时或阻塞的操作，应该将该操作放到另一个线程执行。

* `addScheduledHandler:` 方法用于注册当命令缓冲区被调度时会被调用的代码块。当系统中其他由 `MTLCommandBuffer` 对象或其他 API 提交的工作之间的任何依赖关系得到满足时，命令缓冲区被认为是调度的。我们可以为一个命令缓冲对象注册多个调度处理程序。

* `waitUntilScheduled` 方法会同步等待直到命令缓冲区被调度并且通过 `addScheduledHandler:` 方法注册的所有处理程序都完成之后才返回。

* `addCompletedHandler:` 方法用于注册在设备完成命令缓冲区的执行后被调用的代码块。可以为命令缓冲区注册多个完成处理程序。

* `waitUntilCompleted` 方法会同步等待直到设备完成了该命令缓冲区的执行并且通过 `addCompletedHandler:` 方法注册的所有处理程序都返回以后它才返回。

`presentDrawable:` 方法是完成处理程序的特例。这种方便的方法用于在命令缓冲区被调度时展示可显示资源（`CAMetalDrawable` 对象）的内容。


# 命令编码器

命令编码器是一个临时对象，我们使用它以 GPU 可以执行的格式将命令和状态写入单个命令缓冲区。许多命令编码器对象方法将命令附加到命令缓冲区。当命令编码器处于活动状态时，它有就可以为其命令缓冲区附加命令。完成命令编码后，调用 `endEncoding` 方法。要编写更多命令，可以创建新的命令编码器。

## 创建命令编码器对象

因为命令编码器将命令附加到特定的命令缓冲区中，所以可以通过从要与之一起使用的 `MTLCommandBuffer` 对象来请求创建一个命令编码器。我们可以使用下面的 `MTLCommandBuffer` 对象的方法创建各种类型的命令编码器：

* `renderCommandEncoderWithDescriptor:` 方法为渲染到 `MTLRenderPassDescriptor` 中附件的图形创建一个 `MTLRenderCommandEncoder` 对象。
* `computeCommandEncoder` 方法为数据并行计算创建 `MTLComputCommandEncoder` 对象。
* `blitCommandEncoder` 方法为内存操作创建 `MTLBlitCommandEncoder` 对象。
* `parallelRenderCommandEncoderWithDescriptor:` 方法创建一个 `MTLParallelRenderCommandEncoder` 对象，该对象允许多个 `MTLRenderCommandEncoder` 对象在不同线程上运行，同时仍然渲染到共享的 `MTLRenderPassDescriptor` 中指定的附件上。

## 渲染命令编码器 (Render Command Encoder)

图形渲染可以通过渲染过程来描述。`MTLRenderCommandEncoder` 对象表示与单个渲染过程关联的渲染状态和绘图命令。`MTLRenderCommandEncoder` 需要关联的 `MTLRenderPassDescriptor`，其中包括用作渲染命令目标的颜色，深度和模板附件。`MTLRenderCommandEncoder` 具有以下方法：

* 指定包含顶点，片段或纹理图像数据的图形资源，例如缓冲区和纹理对象。
* 指定一个 `MTLRenderPipelineState` 对象，该对象包含编译的渲染状态，包括顶点和片段着色器。
* 指定固定功能的状态，包括视口，三角形填充模式，剪刀矩形，深度和模板测试以及一些其他的值。
* 绘制 3D 图元。

## 计算命令编码器 (Compute Command Encoder)

对于数据并行计算，`MTLComputeCommandEncoder` 协议提供了对命令缓冲区中的命令进行编码的方法，这些命令可以指定计算函数及其参数（例如，纹理，缓冲区和采样器状态），并分派计算函数以执行。要创建计算命令编码器对象，可以使用 `MTLCommandBuffer` 的 `computeCommandEncoder` 方法。

## Blit 命令编码器 (Blit Command Encoder)

`MTLBlitCommandEncoder` 协议具有在缓冲区（MTLBuffer）和纹理（MTLTexture）之间进行内存复制操作的方法。`MTLBlitCommandEncoder` 协议还提供了用纯色填充纹理和生成 mipmap 的方法。可以使用 `MTLCommandBuffer` 的 `blitCommandEncoder` 方法来创建 Blit 命令编码器对象。

# 多线程、命令缓冲区和命令编码器

大多数应用程序使用单个线程在单个命令缓冲区中对单个帧的渲染命令进行编码。在每个帧的末尾，提交了命令缓冲区，然后就开始调度和执行命令。

如果想要并行地进行命令缓冲区编码，那么可以同时创建多个命令缓冲区，并使用单独的线程对每个命令缓冲区进行编码。如果我们提前知道命令缓冲区的执行顺序，那么 `MTLCommandBuffer` 的 `enqueue` 方法可以在命令队列中声明执行顺序，而无需等待命令编码和提交。否则，当命令缓冲区北被提交时，它会在命令队列中排在它前面的命令缓冲区之后分配一个位置。

一次只能有一个 CPU 线程访问命令缓冲区。一个线程有一个命令缓冲区，那么使用多线程就可以创建多个命令缓冲区。

下图展示了使用三个线程的例子。每个线程拥有自己的命令缓冲区。对于每一个线程，每个命令编码器可以访问到与之关联的命令缓冲区。图中也展示了每个命令缓冲区接受来自不同命令编码器的命令。当我们完成了编码，就调用命令编码器的 `endEncoding` 方法，这时候一个新的命令编码器对象就可以被编码到命令缓冲区了。

<img src="https://developer.apple.com/library/archive/documentation/Miscellaneous/Conceptual/MetalProgrammingGuide/Art/Cmd-Model-threads_2x.png" style="width: 80%;" />

`MTLParallelRenderCommandEncoder` 对象允许在多个命令编码器中分解单个渲染过程，并将其分配给单独的线程。