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
