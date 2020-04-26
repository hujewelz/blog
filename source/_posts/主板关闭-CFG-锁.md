---
title: 关闭主板 CFG 锁
date: 2020-04-04 08:08:41
tags: 
	- 黑苹果 
	- OpenCore
categories: Hackintosh
---

在很多新的主板中都会有 "CFG Locak" 的选项，它的作用是关闭或者开启 MSR 0xe2 寄存器。

<!--more-->

**MSR 0xE2** 是 Model Specific Register 的一个寄存器位数锁定，属于非标准寄存器，是用来控制 CPU 的工作环境和读取工作状态，例如电压、温度、功耗等非程序性指标。而 Mac OS 的电源管理中 CPU 的 P-State (频率) 和 C-State (睡眠) 就是放在 MSR 寄存器里的。现在世面上大多数有 UEFI 的主板厂商默认都锁定了 MSR 寄存器，也就是 BIOS 中的 CFG Lock 是 Disable 的（如果你的主板 BIOS 有这个选项的话）。

如果 CFG Lock 是开启状态（即 MSR 0xE2 是被锁定的），MSR 0xE2 就是只读的，当 AppleIntelCPUPowernamegement 一旦去写入数据，马上就会内核崩溃。当我们使用黑苹果时就必须解锁 MSR 0xE2，否则就无法使用原生电源管理了。

如果你的主板 BIOS 有 "CFG Lock" 选项的话，那么你只需要关闭即可，如果很不幸没有该选项就可以按照下面的方式来关闭 CFG Lock。

## 确认主板 CFG Lock 是否上锁

打开 Hackintool, 选则上面的 Tools（工具）选项，点击下边最左侧的 Intel 图标，输入密码，过几秒钟后就会显示大量内容，搜索 CFG 就能快速找到我们想要的内容，如果是 `CFG Lock............................. : 0 (MSR not locked)` 则表明已经解锁了 MSR，结果是 1 就表明没有解锁。

![](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/cfglock/img1.png)

## 关闭 CFG Lock

在开始下面的工作之前，再强调一遍，如果你的主板 BIOS 有 "CFG Lock" 选项的话，那么你只需要关闭即可。

### 准备工作

在正式开始之前，你需要下载下面的这些内容：

- [VerifyMsrE2](https://github.com/acidanthera/OpenCorePkg/releases)
- [Modifed GRUB Shell](https://github.com/datasone/grub-mod-setup_var/releases)
- [UEFITool](https://github.com/LongSoft/UEFITool/releases) （看清楚名字千万不要下载错了）
- [Universal-IFR-Extractor](https://github.com/LongSoft/Universal-IFR-Extractor/releases)

然后在你的主板提供商的网站上下载对应版本的 BIOS。

### 检查你的主板是否支持关闭 CFG Lock

我们从下载的 OpenCore 包中，在 `EFI/OC/Tools` 可以找到 VerifyMsrE2.efi，通过它我们可以知道主板是否支持关闭 CFG Lock。

将下载好的 VerifyMsrE2.efi 拷贝到你的 `EFI/OC/Tools` 目录下。然后用 ProperTree 打开 `config.plis` , 在 `Misc -> Tools` 中添加 `VerifyMsrE2.efi`。

![](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/cfglock/img3.png)

重启电脑，在 OpenCore 的引导界面中选择刚刚添加的 `VerifyMsrE2`，然后你就知道你的主板是否支持关闭 CFG Lock 了。

### 手动开启

###### 第一步

打开下载好的 UEFITool, 然后将下载好的 BIOS 文件拖到软件中，按下快捷鍵 `Command + F`，在窗口中点击 Text 选项，输入『CFG Lock』后点击 OK。

![Example](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/cfglock/search_cfg_lock.png)

在最下面就会出现 `Unicode text "CFG Lock" found in PE32 image section at offset 85FD0h` 的信息。双击该信息，就会自动定位到包含有 CFG Lock 信息的模块, `PE32 image section`。选中 `PE32 image section`, 然后鼠标右键，选择 Extract body, 填好名称后保存到合适的位置，后缀名不要修改保持默认就好。

![](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/cfglock/cfg_lock_section_info.png)

###### 第二步

在终端使用 `ifrextract` 将刚刚保存的文件转换为文本文件，命令如下:

```shell
path/to/ifrextract path/to/youfile path/to/setup.txt
```

{% note warning %}
上面命令中的路径请自行替换成自己实际的路径
{% endnote %}

###### 第三步

打开刚刚转换成功的文本文件，然后搜索 `CFG Lock, VarStoreInfo (VarOffset/VarName):`，我搜索到的结果是 **CFG Lock, VarStoreInfo (VarOffset/VarName): 0x5C1**，这里的 **0x5C1** 就是我们要找的偏移量, 可以看到后面 `VarStore: 0x1`, 这就代表是锁住的。我们做了这么多就是为了将该地址 (0x5C1) 的数据修改为 `0x0`。

![](https://cdn.jsdelivr.net/gh/hujewelz/CDN-for-myblog/images/cfglock/cfg_lock_address.png)

###### 第四步

将下载好的 `modGRUBShell.efi` 重命名为 `BOOTX64.efi`, 在一个空白的引导 U 盘内创建 `EFI` 文件夹，然后在 `EFI` 文件夹中创建 `BOOT` 文件夹，并将重命名的 `BOOTX64.efi` 放入其中。
如果没有引导 U 盘，可以把 U 盘插入电脑，然后格式化为 `fat32` 格式，命名为 `EFI` 就可以了。

重启电脑，选择从 U 盘启动，这将会启动 grub 模式，然后使用如下命令来解锁 MSR 0xE2:

```shell
setup_var 0x5C1 0x00
```

设置好后你可以通过 `setup_var 0x5C1`， 来查看设置的值有没有成功，如果出现 `offset 0x5C1 is: 0x00`，则表示设置成功。

{% note danger %}
这里的 0x5C1 是笔者的偏移量，请一定要使用自己的。

请注意，可变偏移量不仅对于每个主板都是唯一的，甚至对于其固件版本也是唯一的。未经检查，切勿尝试使用偏移量。
{% endnote %}

## 收尾

完成上面的工作后，你就成功的解锁了 MSR 0xE2。然后就可以将 config.plist 做如下设置：

`Kernel -> Quirk` 选项中：

- `AppleCpuPmCfgLock` 设置为 `False`
- `AppleXcpmCfgLock` 设置为 `False`

`UEFI -> Quirk` 选项中：

- IgnoreInvalidFlexRatio 设置为 `False`

至此关闭主板 CFG Lock 的整个工作就结束了，苹果原生电源管理也能正常工作了。
