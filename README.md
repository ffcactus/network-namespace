# Overview

这个项目利用Linux的network namespace功能实现在一台物理机上模拟两个硬件。具体来说，我们假设：
- 有2个服务器，他们有独立的IP地址。
- 有1个Switch，这2个服务器通过这个Switch与外界的网络通讯。
