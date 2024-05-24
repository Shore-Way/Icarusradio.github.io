---
layout: default
---
# Ubuntu 22.04 下修改驱动使 Intel DG1 可以在 Jellyfin 下解码
## 起因
最近看了皮蛋熊大佬的直通 DG1 教程（[NAS独显转码新王晋级， Intel DG1驱动第二阶段，PVE内核+群晖SA6400驱动分享！](https://www.bilibili.com/video/BV1wb4y1G7kF)）自己也入手了一块 DG1。

但是大佬提供的教程里面只有群晖 SA6400，而我本人不喜欢群晖的系统，希望能够在原生的 Linux 上运行 Jellyfin。于是便有了本篇教程。

## 背景知识
查阅 [Jellyfin 官网教程](https://jellyfin.org/docs/general/administration/hardware-acceleration/intel#known-issues-and-limitations-on-linux)可以得知：

> 6. The kernel support for Intel Gen 12 DG1 is incomplete in upstream. Intel DKMS and custom iHD driver are required.

即 Linux 内核对 DG1 的支持是不完整的。如果需要 Jellyfin 成功解码需要安装 DKMS 和定制的 iHD 驱动，换句话说内核与驱动都需要修改。

Intel 官方开源了暂时没有推送至主线内核的更新：[intel-gpu/intel-gpu-i915-backports](https://github.com/intel-gpu/intel-gpu-i915-backports)。这个可以不用编译，因为 Intel 官方维护了一个 Ubuntu 仓库提供编译好的软件包。不过推荐安装之前先查阅一下这个 GitHub 仓库，原因后面会提及。

查阅 Intel 官网可以得知驱动分三个版本：Rolling, Production 和 Long Term Support (LTS)。如果追求新的特性选择 Rolling 版本，追求稳定则选择 Production 或者 LTS 版本。LTS 相比 Production 可以提供长达三年的支持。

每个版本具体发行信息可以在 [Intel 的发行信息](https://dgpu-docs.intel.com/releases/index.html)里查看。点开每个版本发行信息，可以得知在 Ubuntu 22.04 下，DG1 在三个版本均测试通过。

## 准备工作
### 安装前注意事项
如果是物理机安装，首先请参阅皮蛋熊大佬的教程修改 BIOS 来支持点亮 DG1。如果无法点亮 DG1 请先解决这个问题，本篇教程不涉及解决此问题。确认可以点亮后，请拔下 DG1，使用 CPU 自带的核显或者其他亮机卡完成安装。

如果是虚拟机安装，首先请参阅皮蛋熊大佬的教程修改 PVE 内核来支持直通 DG1。设置完成后创建虚拟机，先不要直通 DG1，等设置完成后再直通。

### 安装 Ubuntu 22.04 LTS
上述设置完成后，物理机和虚拟机系统安装方式类似，本教程以虚拟机安装为例。

推荐安装 server 版本，本教程以 server 版为例，desktop 版本遇到的额外问题后面会单独描述。

### 添加 Intel 官方 Ubuntu 仓库
首先先添加 Intel 的 GPG 密钥：
```bash
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | sudo gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg
```

添加 LTS 仓库（与下面 Rolling 仓库二选一）：

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy/lts/2350 unified" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list
```

添加 Rolling 仓库（与上面 LTS 仓库二选一）：

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy client" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list
```

更新仓库：
```bash
sudo apt update
```

## 安装 DKMS 和定制的 iHD 驱动
### 安装 DKMS
**防杠声明**：理论上 DKMS 并不需要安装指定内核，每当有新内核安装时会自动编译模块。但是本教程为了保证成功，安装了 Intel 测试成功的内核版本。本教程无法保证在不安装对应内核版本下也能正常工作。后续更新 DKMS 也推荐将内核一同更新。

在安装之前，先用 `apt` 指令查看仓库内版本

```bash
apt-cache madison intel-i915-dkms
```

这里以 Rolling 仓库为例，目前（2024 年 5 月 24 日）可查询到的版本为：

```
intel-i915-dkms | 1.24.1.11.240117.14+i16-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.24.1.11.240117.14+i16-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
intel-i915-dkms | 1.23.9.11.231003.15+i19-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.9.11.231003.15+i19-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
intel-i915-dkms | 1.23.8.20.230810.22+i33-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.8.20.230810.22+i33-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
intel-i915-dkms | 1.23.7.17.230608.24+i37-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.7.17.230608.24+i37-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
intel-i915-dkms | 1.23.6.24.230425.29+i44-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.6.24.230425.29+i44-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
intel-i915-dkms | 1.23.5.19.230406.21.5.17.0.1034+i38-1 | https://repositories.intel.com/gpu/ubuntu jammy/client amd64 Packages
intel-i915-dkms | 1.23.5.19.230406.21.5.17.0.1034+i38-1 | https://repositories.intel.com/gpu/ubuntu jammy/client i386 Packages
```

即最新版本为 `1.24.1.11.240117.14+i16-1`，记住这个版本。打开 [intel-gpu/intel-gpu-i915-backports](https://github.com/intel-gpu/intel-gpu-i915-backports) 这个仓库。在上方标签里搜索“24.1.11”找到对应标签并点击。然后点击 `versions` 文件，查看测试对应的内核版本，比如这个标签的文件内容如下：

```
BACKPORTS_RELEASE_TAG="I915_24.1.11_PSB_240117.14"
DII_KERNEL_TAG="I915-24.1.11"
DII_KERNEL_HEAD="b204f78969214"
UBUNTU_22.04_SERVER_KERNEL_VERSION="5.15.0-100"
UBUNTU_22.04_DESKTOP_KERNEL_VERSION="6.5.0-25"
VANILLA_5.15LTS_KERNEL_VERSION="5.15.150"
VANILLA_6.1LTS_KERNEL_VERSION="6.1.80"
SLES15_SP4_KERNEL_VERSION="5.14.21-150400.24.108"
SLES15_SP5_KERNEL_VERSION="5.14.21-150500.55.49"
RHEL_9.0_KERNEL_VERSION="5.14.0-70.30.1"
RHEL_9.2_KERNEL_VERSION="5.14.0-284.30.1"
RHEL_9.3_KERNEL_VERSION="5.14.0-362.13.1"
RHEL_8.9_KERNEL_VERSION="4.18.0-513.5.1"
RHEL_8.8_KERNEL_VERSION="4.18.0-477.13.1"
RHEL_8.6_KERNEL_VERSION="4.18.0-372.32.1"
VANILLA_5.10LTS_KERNEL_VERSION="5.10.211"
VANILLA_5.4LTS_KERNEL_VERSION="5.4.270"
```

这里我们关注 `UBUNTU_22.04_DESKTOP_KERNEL_VERSION`，即测试验证的内核版本为 `6.5.0-25`。安装对应的内核版本：

```bash
sudo apt install linux-image-6.5.0-25-generic linux-headers-6.5.0-25-generic linux-modules-6.5.0-25-generic linux-modules-extra-6.5.0-25-generic
```

如果安装的是 server 版本，安装完成后重启即可，因为 server 版默认安装 5.15 内核，启动时会自动选择 6.5 内核进行启动。Desktop 版本默认安装 6.5 内核，可能版本号会比这个大，需要修改 GRUB 来指定启动使用的内核。

重启完成后输入 `uname -a` 来确认启动内核是否为安装版本。确认成功后，安装 DKMS 和 GPU 固件。

```bash
sudo apt install intel-i915-dkms intel-fw-gpu
```

安装完成后再次重启，输入 `sudo dmesg | grep -i backport` 来确认内核是否成功安装。如果出现类似如下结果说明安装成功。

```
[    5.963854] COMPAT BACKPORTED INIT
[    5.968761] Loading modules backported from I915-23.6.24
[    5.976154] Backport generated by backports.git I915_23.6.24_PSB_230425.29
[    6.069699] [drm] I915 BACKPORTED INIT
```

此时关闭机器。物理机装上 DG1，如果使用核显安装系统，请 BIOS 内选择仅独立显卡工作来关闭核显；如果使用亮机卡安装系统，请拆下亮机卡。虚拟机上直通 DG1，并将显示关闭。操作完成后打开机器。

### 安装 Jellyfin
Ubuntu 下安装 Jellyfin 有两种方式，一种是通过 apt 安装官方编译的 deb 包，另一种是使用 docker。这里选择 docker 安装，镜像选择 nyanmisaka 大佬制作的 Jellyfin 中国特供版。

首先查看自己用户和用户组的 ID，命令行输入 `id` 指令，查看显示的 `uid` 和 `gid` 的对应值，这里以两个值均是 1000 为例，请修改为自己机器上的数值。

再查看显卡对应用户组，输入 `ls -l /dev/dri/` 指令，查看 `renderD128` 对应的用户组。这里显示为 `render`。

```
total 0
drwxr-xr-x  2 root root         80  5月 24 00:23 by-path
crw-rw----+ 1 root video  226,   0  5月 24 00:23 card0
crw-rw----+ 1 root render 226, 128  5月 24 00:23 renderD128
```

然后输入指令 `getent group render | cut -d: -f3` 查看 `render` 组对应的 ID。这里以 ID 是 110 为例，请修改为自己机器上的数值。

管理 Jellyfin 的配置文件有两种方式：
1. 本地创建文件夹，这样可以更方便的查看日志文件
2. 使用 docker 卷管理，这样可以交给 docker 管理

#### 本地创建文件夹
输入指令 `mkdir -p ~/Jellyfin/{config,cache}` 在自己的用户目录创建配置和缓存文件夹

然后创建 docker 容器（指令需要注意和修改的地方参阅下方注意事项）：

```bash
sudo docker run -d \
 --name=jellyfin \
 --mount type=bind,source=~/Jellyfin/config,target=/config \
 --mount type=bind,source=~/Jellyfin/cache,target=/cache \
 --mount type=bind,source=/path/to/media,target=/media \
 --user 1000:1000 \
 --group-add="110" \
 --net=host \
 --restart=unless-stopped \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 nyanmisaka/jellyfin
```

#### 使用 docker 卷管理
先创建两个 docker 卷分别存储 Jellyfin 的配置文件与缓存：

```bash
sudo docker volume create jellyfin-config
sudo docker volume create jellyfin-cache
```

然后创建 docker 容器（指令需要注意和修改的地方参阅下方注意事项）：

```bash
sudo docker run -d \
 --name=jellyfin \
 --volume jellyfin-config:/config \
 --volume jellyfin-cache:/cache \
 --mount type=bind,source=/path/to/media,target=/media \
 --user 1000:1000 \
 --group-add="110" \
 --net=host \
 --restart=unless-stopped \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 nyanmisaka/jellyfin
```

#### 注意事项
1. 请将 `--user 1000:1000` 里数值修改为自己用户对应的 ID。
2. 请将 `--group-add="110"` 里数值修改为查询到的用户组 ID。
3. 请将 `--mount type=bind,source=/path/to/media,target=/media` 中 `/path/to/media` 修改为存储媒体的路径，并确保自己用户对这个文件夹有修改权限。

### 安装定制 iHD 驱动
安装完成 Jellyfin 后，输入指令进入 docker 内命令行：

``` bash
sudo docker exec -it jellyfin /bin/bash
```

在命令行内输入指令测试 ffmpeg：

```bash
/usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```

此时应该会报错，因为我们还没有替换驱动文件，docker 内自带的文件不支持 DG1。这里我编译好了驱动文件，请去[这个仓库](https://github.com/Icarusradio/intel-dg1-driver)的 Releases 中下载。这个仓库我会尽量保持与 Intel 官方最新版本一致。如果想要自行编译，请参阅文章附录的编译教程。

用以下指令将下载的 `iHD_drv_video.so` 替换 docker 内的：

```bash
sudo docker cp iHD_drv_video.so jellyfin:/usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
```

此时再输入上述指令进入 docker 内命令行并执行测试 ffmpeg 指令，如果出现类似输出证明替换成功：

```
Trying display: drm
libva info: VA-API version 1.21.0
libva info: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_21
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.21 (libva 2.21.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 24.1.5 (8068c2e)
vainfo: Supported profile and entrypoints
      VAProfileNone                   :	VAEntrypointVideoProc
      VAProfileNone                   :	VAEntrypointStats
      VAProfileMPEG2Simple            :	VAEntrypointVLD
      VAProfileMPEG2Simple            :	VAEntrypointEncSlice
      VAProfileMPEG2Main              :	VAEntrypointVLD
      VAProfileMPEG2Main              :	VAEntrypointEncSlice
      VAProfileH264Main               :	VAEntrypointVLD
      VAProfileH264Main               :	VAEntrypointEncSlice
      VAProfileH264Main               :	VAEntrypointFEI
      VAProfileH264Main               :	VAEntrypointEncSliceLP
      VAProfileH264High               :	VAEntrypointVLD
      VAProfileH264High               :	VAEntrypointEncSlice
      VAProfileH264High               :	VAEntrypointFEI
      VAProfileH264High               :	VAEntrypointEncSliceLP
      VAProfileVC1Simple              :	VAEntrypointVLD
      VAProfileVC1Main                :	VAEntrypointVLD
      VAProfileVC1Advanced            :	VAEntrypointVLD
      VAProfileJPEGBaseline           :	VAEntrypointVLD
      VAProfileJPEGBaseline           :	VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline:	VAEntrypointVLD
      VAProfileH264ConstrainedBaseline:	VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline:	VAEntrypointFEI
      VAProfileH264ConstrainedBaseline:	VAEntrypointEncSliceLP
      VAProfileHEVCMain               :	VAEntrypointVLD
      VAProfileHEVCMain               :	VAEntrypointEncSlice
      VAProfileHEVCMain               :	VAEntrypointFEI
      VAProfileHEVCMain               :	VAEntrypointEncSliceLP
      VAProfileHEVCMain10             :	VAEntrypointVLD
      VAProfileHEVCMain10             :	VAEntrypointEncSlice
      VAProfileHEVCMain10             :	VAEntrypointEncSliceLP
      VAProfileVP9Profile0            :	VAEntrypointVLD
      VAProfileVP9Profile0            :	VAEntrypointEncSliceLP
      VAProfileVP9Profile1            :	VAEntrypointVLD
      VAProfileVP9Profile1            :	VAEntrypointEncSliceLP
      VAProfileVP9Profile2            :	VAEntrypointVLD
      VAProfileVP9Profile2            :	VAEntrypointEncSliceLP
      VAProfileVP9Profile3            :	VAEntrypointVLD
      VAProfileVP9Profile3            :	VAEntrypointEncSliceLP
      VAProfileHEVCMain12             :	VAEntrypointVLD
      VAProfileHEVCMain12             :	VAEntrypointEncSlice
      VAProfileHEVCMain422_10         :	VAEntrypointVLD
      VAProfileHEVCMain422_10         :	VAEntrypointEncSlice
      VAProfileHEVCMain422_12         :	VAEntrypointVLD
      VAProfileHEVCMain422_12         :	VAEntrypointEncSlice
      VAProfileHEVCMain444            :	VAEntrypointVLD
      VAProfileHEVCMain444            :	VAEntrypointEncSliceLP
      VAProfileHEVCMain444_10         :	VAEntrypointVLD
      VAProfileHEVCMain444_10         :	VAEntrypointEncSliceLP
      VAProfileHEVCMain444_12         :	VAEntrypointVLD
      VAProfileHEVCSccMain            :	VAEntrypointVLD
      VAProfileHEVCSccMain            :	VAEntrypointEncSliceLP
      VAProfileHEVCSccMain10          :	VAEntrypointVLD
      VAProfileHEVCSccMain10          :	VAEntrypointEncSliceLP
      VAProfileHEVCSccMain444         :	VAEntrypointVLD
      VAProfileHEVCSccMain444         :	VAEntrypointEncSliceLP
      VAProfileAV1Profile0            :	VAEntrypointVLD
      VAProfileHEVCSccMain444_10      :	VAEntrypointVLD
      VAProfileHEVCSccMain444_10      :	VAEntrypointEncSliceLP
```

再输入以下指令检查 OpenCL。

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -v verbose -init_hw_device vaapi=va:/dev/dri/renderD128 -init_hw_device opencl@va
```

如果出现类似输出证明成功：

```
ffmpeg version 6.0.1-Jellyfin Copyright (c) 2000-2023 the FFmpeg developers
  built with gcc 12 (Debian 12.2.0-14)
  configuration: --prefix=/usr/lib/jellyfin-ffmpeg --target-os=linux --extra-version=Jellyfin --disable-doc --disable-ffplay --disable-ptx-compression --disable-static --disable-libxcb --disable-sdl2 --disable-xlib --enable-lto --enable-gpl --enable-version3 --enable-shared --enable-gmp --enable-gnutls --enable-chromaprint --enable-opencl --enable-libdrm --enable-libass --enable-libfreetype --enable-libfribidi --enable-libfontconfig --enable-libbluray --enable-libmp3lame --enable-libopus --enable-libtheora --enable-libvorbis --enable-libopenmpt --enable-libdav1d --enable-libsvtav1 --enable-libwebp --enable-libvpx --enable-libx264 --enable-libx265 --enable-libzvbi --enable-libzimg --enable-libfdk-aac --arch=amd64 --enable-libshaderc --enable-libplacebo --enable-vulkan --enable-vaapi --enable-amf --enable-libvpl --enable-ffnvcodec --enable-cuda --enable-cuda-llvm --enable-cuvid --enable-nvdec --enable-nvenc
  libavutil      58.  2.100 / 58.  2.100
  libavcodec     60.  3.100 / 60.  3.100
  libavformat    60.  3.100 / 60.  3.100
  libavdevice    60.  1.100 / 60.  1.100
  libavfilter     9.  3.100 /  9.  3.100
  libswscale      7.  1.100 /  7.  1.100
  libswresample   4. 10.100 /  4. 10.100
  libpostproc    57.  1.100 / 57.  1.100
[AVHWDeviceContext @ 0x64cafd64e2c0] libva: VA-API version 1.21.0
[AVHWDeviceContext @ 0x64cafd64e2c0] libva: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
[AVHWDeviceContext @ 0x64cafd64e2c0] libva: Found init function __vaDriverInit_1_21
[AVHWDeviceContext @ 0x64cafd64e2c0] libva: va_openDriver() returns 0
[AVHWDeviceContext @ 0x64cafd64e2c0] Initialised VAAPI connection: version 1.21
[AVHWDeviceContext @ 0x64cafd64e2c0] VAAPI driver: Intel iHD driver for Intel(R) Gen Graphics - 24.1.5 (8068c2e).
[AVHWDeviceContext @ 0x64cafd64e2c0] Driver not found in known nonstandard list, using standard behaviour.
[AVHWDeviceContext @ 0x64cafd67c3c0] 0.0: Intel(R) OpenCL Graphics / Intel(R) UHD Graphics 770
[AVHWDeviceContext @ 0x64cafd67c3c0] Intel QSV to OpenCL mapping function found (clCreateFromVA_APIMediaSurfaceINTEL).
[AVHWDeviceContext @ 0x64cafd67c3c0] Intel QSV in OpenCL acquire function found (clEnqueueAcquireVA_APIMediaSurfacesINTEL).
[AVHWDeviceContext @ 0x64cafd67c3c0] Intel QSV in OpenCL release function found (clEnqueueReleaseVA_APIMediaSurfacesINTEL).
Hyper fast Audio and Video encoder
usage: ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...

Use -h to get full help or, even better, run 'man ffmpeg'
```

## 附录
### 编译 iHD 驱动
开源驱动位于 [intel/media-driver](https://github.com/intel/media-driver)，这里说明了如何编译支持 DG1 的驱动：

> Media-driver requires special i915 kernel mode driver (KMD) version to support the following platforms since upstream version of i915 KMD does not fully support them(pending patches upstream). To enable these platforms, it requires to specify ENABLE_PRODUCTION_KMD=ON (default: OFF) build configuration option.  
>   - DG1/SG1
>   - ATSM
