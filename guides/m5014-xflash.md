---
layout: default
---
# IBM M5014/M5015 刷 LSI 固件教程

**声明：刷固件有风险，请认真阅读本教程，如果出现误操作导致变砖本人概不负责。**

## 刷固件理由
IBM 原版固件启动太慢了，从启动到加载完成需要好几分钟，而 LSI 的固件启动比较快。实际上 M5014/M5015 就是 LSI 9260-8i 的 OEM 版，在原版上阉割了一些功能。

## 准备工作
### 下载软件
#### MegaRec
刷机主要用的软件，HP 官网有下载。前往 [HP 企业官网](https://support.hpe.com/)，搜索“LSI MEGARAID CACHE CLEARING”  
![hpe-search](https://user-images.githubusercontent.com/8037656/227863251-61cd0a33-566a-4e67-b83a-0321b952a1b0.png)  
搜索结果选择下图所示，点击下载即可  
![hpe-megarec](https://user-images.githubusercontent.com/8037656/227863260-8a5b1d3b-e952-48b4-be36-7402302b5bd2.png)

#### MegaCLI（可选）
主要用来查看控制器对应编号，如果只有一张卡可以不下载。如果有多张卡或者为了保险起见可以下载。  
LSI 目前被博通收购了，所以要去[博通官网](https://www.broadcom.com/)下载。进入官网，如下图所示，选择 “Support Documents and Downloads”。  
![broadcom](https://user-images.githubusercontent.com/8037656/227862862-91e0e72e-5851-4996-bf33-f3b313e7200c.png)  
在搜索框输入“MegaCLI”，点击搜索  
![broadcom-search](https://user-images.githubusercontent.com/8037656/227862877-1f5fa7e3-5ee4-47f4-9894-316afac8156e.png)  
在“Management Software and Tools”里选择第一个下载。新页面拖到最下面点我同意就可以下载了。  
![broadcom-megacli](https://user-images.githubusercontent.com/8037656/227862873-256591ac-be33-4f93-866d-2daee3be277f.png)

#### LSI 9260-8i 固件
同 MegaCLI 一样，也要在博通官网下载。  
在搜索页面“Product Group”选择“Legacy Products”，“Product Family”选择“All Legacy Products”，“Product Name”输入 9260 并选择“MegaRAID SAS 9260-8i”，点击“Search”搜索。  
![broadcom-9260](https://user-images.githubusercontent.com/8037656/227862869-abedd3c6-dc31-4c13-9c54-2aba8c03dfd4.png)  
固件在“Firmware”内。“Current”为最新固件，“Archive”为历史固件。  
![broadcom-fw](https://user-images.githubusercontent.com/8037656/227862871-c33bf98d-c18c-477b-87fc-cec8bd3086b7.png)

#### LSI SBR 固件
SBR 固件是卡的启动固件，需要刷入正确的固件才能成功启动卡。外国论坛有人整理好 9260-8i 和 M5014 等卡的 SBR 固件：[SAS2108 (LSI 9260) based firmware files](https://forums.laptopvideo2go.com/topic/29166-sas2108-lsi-9260-based-firmware-files/)。

#### Rufus
Rufus 是用来制作 DOS 启动盘的软件，[官网](https://rufus.ie/)。

### 制作 DOS 启动盘
准备一个 U 盘，**备份里面的数据**，因为会格式化 U 盘。U 盘容量超过 200 MB 即可。  
打开 Rufus，插入 U 盘。设备选择你插入的 U 盘，引导类型选择 FreeDOS，其他保持默认，点击开始。  
![rufus](https://user-images.githubusercontent.com/8037656/227862711-13bf83bd-2735-44b7-bd91-3f901fdc8b6b.png)  
将 SBR 固件全部解压至 U 盘。新建两个文件夹：`tools` 和 `lsi`。  
解压 MegaRec 的压缩包，将 `MegaRec.exe` 和 `DOS4GW.EXE` 放入 `tools` 文件夹。  
解压 MegaCLI 的压缩包，将 `DOS` 文件夹下的 `MegaCLI.exe` 放入 `tools` 文件夹。  
解压下载的 LSI 固件压缩包，将 `mr2108fw.rom` 放入 `lsi` 文件夹。如果下载了多个版本，可以以版本号最后 4 位数字重命名，比如 `0189.rom`。

## 刷入 LSI 固件
### 进入 DOS 模式
进入主板 BIOS，开启 CSM 模式。插入制作好的 DOS 启动盘，开机时选择 U 盘启动进入 DOS 模式。  
注意，最好用旧主板，新的主板不支持 Legacy 模式启动下显示。实测 Intel 6 代及配套主板可以支持。  
系统启动后，输入 `dir` 列出当前目录下内容，输出内容应该与下图类似  
![dos-dir](https://user-images.githubusercontent.com/8037656/227862888-5dd4635c-84b2-4c56-aa3b-4271cef0f15b.jpg)  
输入 `cd tools` 进入 `tools` 文件夹。如果觉得屏幕上内容太多，可以用 `cls` 指令清屏。

### 利用 MegaCLI 确定控制器编号（可选）
如果只有一张卡可以跳过这步，因为一般编号都是 0。如果想确定是不是 0，可以用 MegaCLI 确认。  
输入 `megacli.exe -AdpAllInfo -aAll>all.txt`，待指令执行完毕按电源键关机。在系统内查看 U 盘内 `tools` 文件夹内的 `all.txt`，应该有如下类似输出：  
```
Adapter #0

==============================================================================
                    Versions
                ================
Product Name    : LSI MegaRAID SAS 9260-8i
Serial No       : SV22816038
FW Package Build: 12.12.0-0048
```
Adapter 后面的数字就是对应的控制器编号，这里就是 0。

### 刷入 SBR 固件
先写入空文件清除原有固件：  
```
megarec.exe -writesbr 0 ..\2108\sbrempty.bin
```
如果之前 MegaCLI 显示控制器编号不是 0，那这里要改成对应编号，后面也一样。  
再写入 LSI 的 SBR 固件：
```
megarec.exe -writesbr 0 ..\2108\sbr9260.bin
```
写入完成后输出应该类似下图：  
![dos-sbr](https://user-images.githubusercontent.com/8037656/227867258-8db90545-55a4-490c-831e-ded930bbe2b3.jpg)

### 擦除闪存
输入如下指令：
```
megarec.exe -cleanflash 0
```
同理，这里 0 是 MegaCLI 返回的控制器编号。输出应该类似下图：  
![dos-cleanflash](https://user-images.githubusercontent.com/8037656/227864250-73c94f51-2053-44a1-abc1-efeb41c88343.jpg)  
擦除闪存成功后按下“Ctrl+Alt+Del”重启。这步很关键，**没有重启有可能会变砖**。  
重启后再次进入 DOS 模式，输入 `cd tools` 进入 `tools` 文件夹。

### 选择 LSI 固件版本
虽然 LSI 固件有很多版本，但是有的版本刷入后，并不会被识别成 9260-8i，仍然是 M5014，启动时间还是很慢。  
国外论坛有人整理了哪些版本是可以成功的（[链接](https://forums.servethehome.com/index.php?threads/cross-flashing-an-ibm-m5014-to-lsi-9260.836/post-129624)）：

|固件后 4 位|结果|
|:-:|:-:|
|0102|√|
|0111|√|
|0124|×|
|0139|√|
|0151|√|
|0154|√|
|0167|√|
|0189|√|
|0205|×|
|0239|×|

### 写入 LSI 固件
这里以 0189 固件为例，输入以下指令刷入：
```
megarec.exe -m0flash 0 ..\lsi\0189.ROM
```
同理，这里 0 是 MegaCLI 返回的控制器编号，但是不要修改 `-m0flash`。耐心等待刷入完成  
![dos-m0flash](https://user-images.githubusercontent.com/8037656/227862890-1df49b17-6438-4be8-9cba-723a15cfb399.jpg)  
刷入完成后应该会有“Success”提示  
![](../img/dos-success.jpg)

## 验证
刷入 LSI 固件后，按下“Ctrl+Alt+Del”重启，如果卡型号显示 9260-8i 说明刷入成功  
![](../img/success.jpg)  
刷入新固件后第一次启动初始化会很久，这是正常现象，请耐心等待。  
待初始化完成后进入系统再次重启，看启动时间是否缩短。

## 参考
[How to Flash IBM ServeRAID M5014 to LSI 9260-8i Firmware](https://www.servethehome.com/flash-ibm-serveraid-m5014-lsi-92608i-firmware/)
