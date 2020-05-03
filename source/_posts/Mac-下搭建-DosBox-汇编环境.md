---
title: Mac 下搭建 DOSBox 汇编环境
date: 2019-02-03 19:04:03
tags: 汇编
categories: 汇编
---

DOSBox 是一个创建与 MS-DOS 兼容环境的模拟器, 在 macOS 中我们需要用它来模拟 DOS 环境。

<!--more-->

## 安装

我们可以从 [这里](https://sourceforge.net/projects/dosbox/files/dosbox/0.74-3/DOSBox-0.74-3-1.dmg/download) 下载 DOSBox 最新版本，双击打开安装包，将 DOSBox 拖人应用程序文件目录即可。

{% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/dosbox/1.jpg 600 %}

然后从 [这里](https://pan.baidu.com/s/11yEKsl3_pKx7k9IH-K7mRw) 下载 debug、edit、link、masm 等汇编工具。

> 提取码: uwf1

## 配置 DosBox

打开安装好的 DosBox 默认是在 Z 盘目录下，这里我们需要挂载自己的目录，我们可以在任何地方创建自己的目录，然后加上面下载好的汇编工具放到该目录下，在 DOS 中输入如下命令：

```dos
Z:\>mount c your/path
```

上面的命令将我们的目录挂载到了 c 盘。我们可以通过 `c:` 命令进入到 c 盘下，然后执行 `dir` 命令可以看到 c 盘下的内容了，即我们刚刚挂载的目录里的汇编工具。

{% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/dosbox/3.jpg 600 %}

## 开始汇编

紧接着刚刚的操作，在 c 盘下，我们输入 `debug` 命令并回车，就进入了 debug 模式。

在 debug 模式下，输入 `r` 就可以查看 CPU 寄存器的内容了：

{% img https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/dosbox/4.jpg 600 %}

Debug 是 DOS、Windows 都提供的实模式程序的调试工具。使用它可以查看 CPU 各种寄存器中的内容、内存的情况和在机器码级跟踪程序的运行。

我们常用的 Debug 功能如下：

- **R** 命令查看、改变 CPU 寄存器的内容；
- **D** 命令查看内存中的内容；
- **E** 命令改写内存中的内容；
- **U** 命令将内存中的机器码指令翻译汇编指令；
- **T** 命令执行一条机器指令；
- **A** 命令以汇编指令的格式在内存中写入一条机器指令。
