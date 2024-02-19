---
layout: default
---
# 修改 BIOS 增加 GVT-g 可用 GPU

## 起因
最近在折腾自己搭建的 Proxmox VE 服务器，配置如下：
- 硬件：10900T ES (QTB0) + MSI H410I
- 系统： Proxmox VE 7.x

在研究显卡直通的时候学习到有两种模式：GVT-d 和 GVT-g。最开始研究的时候参考了 [ProxmoxVE-7.0-DIY](https://github.com/xiangfeidexiaohuo/ProxmoxVE-7.0-DIY) 这个仓库。但是遇到了以下问题：
1. 提供的工具来源不明，不知道从何处下载。
2. 教程所提供的步骤在我的配置上无法成功。

后来继续搜索查找到一篇 [CSDN 博客](https://blog.csdn.net/w670165403/article/details/110943507)，在一定程度上解决了第一个问题，但是按照该博客提供的步骤仍无法成功，于是便有了本篇教程。

## 准备工作
### 下载软件
#### 需要修改主板的 BIOS 文件
自行去主板官网下载。本篇教程以 MSI H410I 为例，最新版本为 7C86v24。

#### UEFITool
[UEFITool](https://github.com/LongSoft/UEFITool) 是用来查看 BIOS 文件内设置的软件，目前最新版本是 A68 ([UEFITool_NE_A68_win64.zip](https://github.com/LongSoft/UEFITool/releases/download/A68/UEFITool_NE_A68_win64.zip))。

#### IFR Extractor
[IFR Extractor](https://github.com/LongSoft/Universal-IFR-Extractor) 可以将提取出的 BIOS 内设置转换成可读的文本形式。Windows 最新版本是 0.3.6 ([IRFExtractor_0.3.6_win.zip](https://github.com/LongSoft/Universal-IFR-Extractor/releases/download/v0.3.6/IRFExtractor_0.3.6_win.zip))。

#### 添加“setup_var”支持的 GRUB
给 GRUB 添加了 `setup_var` 指令，用来修改一些隐藏的 BIOS 选项。最新版基于 GRUB 2.06 修改 ([modGRUBShell.efi](https://github.com/datasone/grub-mod-setup_var/releases/download/1.4/modGRUBShell.efi))。

#### RU.EXE（可选）
有些品牌机主板可能会有特殊保护，可以用这个软件直接使用图形化界面修改 BIOS 选项。常见的 DIY 品牌主板不需要用这个软件修改。下载详细步骤请查看[作者的博客](https://ruexe.blogspot.com/)。

#### Ventoy
[Ventoy](https://www.ventoy.net/cn/index.html) 是一个制作可启动 U 盘的开源工具。目前最新版本是 1.0.97 ([ventoy-1.0.97-windows.zip](https://github.com/ventoy/Ventoy/releases/download/v1.0.97/ventoy-1.0.97-windows.zip))。

## 修改原理
Intel GVT-g 是显卡虚拟化技术，有别于 PCIe 设备直通，GVT-g 可以虚拟出多个虚拟的 GPU，从而有效地在虚拟机中提供接近宿主机的图形性能，并且仍然让主机正常使用虚拟化的 GPU。简单的说就是把一个显卡拆成好多个分别给不同虚拟机使用。

为了实现更多 vGPU，需要给核显分配更多的显存。但是绝大多数主板 BIOS 并没有提供调节显存大小的选项。去 Intel 官网翻了一下，发现 Intel NUC 上的 BIOS 有个 `Intel aperture size` 的选项，用于调节显存大小。于是只要想办法修改自己主板BIOS里面的这个设置值即可。

## 修改步骤
### 1. 使用 UEFITool 找到设置对应的模块
用 UEFITool 打开下载的 BIOS 文件。这里以 MSI H410I 为例，打开 UEFITool，用 `Ctrl+O` 打开下载好的 BIOS 文件。如果没有找到，记得选择所有后缀名的文件。

![UEFITool-open](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/62c47861-b01a-4d8e-822c-8ebdb97ecebc)

打开后用 `Ctrl+F` 进行搜索，选择 `Text` 选项卡，然后搜索框内输入 `aperture size`。

![UEFITool-search-1](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/c6a2b06c-b33f-4ab4-a0c8-998b5204ccb9)

在下方搜索结果中双击一个，会跳转到对应模块位置。用 `Ctrl+Shift+E` 提取模块并保存。

![UEFITool-search-2](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/0a172110-c041-42df-8eec-2919b648977b)
![UEFITool-save](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/420d2bb2-09c0-4fc9-8b90-8dd32978497d)

### 2. 找到设置对应的偏移地址
运行 IFR Extractor，如果打开提示如下错误，请安装 [Visual C++ Redistributable Packages for Visual Studio 2013](https://www.microsoft.com/en-us/download/details.aspx?id=40784)。

![MSVCR120](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/d1b10bbd-2975-4bf5-8190-400e14fa05fa)
![MSVCP120](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/de2b2b8e-6ccc-4e23-8f36-d5fc2430121f)

x86 和 x64 两个版本都需要安装。如果有安装 `winget` 可以用以下指令安装。

```
winget install Microsoft.VCRedist.2013.x86 Microsoft.VCRedist.2013.x64
```

用 IFR Extractor 读取刚刚提取出的模块。

![IFRExtractor-open](https://github.com/Icarusradio/Icarusradio.github.io/assets/8037656/2d83932e-0c7e-425d-bace-5ca192840557)

点击 Extract 后保存。打开保存的文本文件，搜索 `aperture`。

```
0x4F70C 		One Of: Aperture Size, VarStoreInfo (VarOffset/VarName): 0x44, VarStore: 0x17, QuestionId: 0x2753, Size: 1, Min: 0x0, Max 0xF, Step: 0x0 {05 91 9F 0F A0 0F 53 27 17 00 44 00 14 10 00 0F 00}
0x4F71D 			One Of Option: 128MB, Value (8 bit): 0x0 {09 07 A1 0F 00 00 00}
0x4F724 			One Of Option: 256MB, Value (8 bit): 0x1 (default) {09 07 A2 0F 30 00 01}
0x4F72B 			One Of Option: 512MB, Value (8 bit): 0x3 {09 07 A3 0F 00 00 03}
0x4F732 			One Of Option: 1024MB, Value (8 bit): 0x7 {09 07 A4 0F 00 00 07}
0x4F739 			One Of Option: 2048MB, Value (8 bit): 0xF {09 07 A5 0F 00 00 0F}
0x4F740 		End One Of {29 02}
```

记住这里两个重要参数：`VarStoreInfo` 指偏移，而 `VarStore` 指设置存放在哪里。回到文件开头，查看 `0x17` 对应的设置名。

```
0x4145A 	VarStore: VarStoreId: 0x16 [B08F97FF-E6E8-4193-A997-5E9E9B0ADB32], Size: 0x10, Name: CpuSetupSgxEpochData {24 2B FF 97 8F B0 E8 E6 93 41 A9 97 5E 9E 9B 0A DB 32 16 00 10 00 43 70 75 53 65 74 75 70 53 67 78 45 70 6F 63 68 44 61 74 61 00}
0x41485 	VarStore: VarStoreId: 0x17 [72C5E28C-7783-43A1-8767-FAD73FCCAFA4], Size: 0x37E, Name: SaSetup {24 1E 8C E2 C5 72 83 77 A1 43 87 67 FA D7 3F CC AF A4 17 00 7E 03 53 61 53 65 74 75 70 00}
0x414A3 	VarStore: VarStoreId: 0x18 [4570B7F1-ADE8-4943-8DC3-406472842384], Size: 0x670, Name: PchSetup {24 1F F1 B7 70 45 E8 AD 43 49 8D C3 40 64 72 84 23 84 18 00 70 06 50 63 68 53 65 74 75 70 00}
```

这里对应设置名为 `SaSetup`。我们需要修改的 `Aperture Size` 位于 `SaSetup` 中，偏移量为 `0x44`。默认值为 `0x1`，即 256MB。如果想修改成更大则需要修改这个数值。

### 3. 安装 Ventoy 并准备启动 GRUB

### 4. 启动 GRUB 并修改对应值，实现增加虚拟 GPU

### 5. 启动 RU.EXE 并修改对应值，实现增加虚拟 GPU（可选）

### 6. 进入 Proxmox VE 验证
