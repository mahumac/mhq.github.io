---
title: 在Windows上用Wireshark抓取带有VLAN标记的报文
date: 2022-03-09 +0800
categories: [Windows]
description: 
pin: false
tags: [Windows, Wireshark, Vlan] 
---



有朋友问，使用Windows PC去抓取交换机SPAN过来的流量，但是无论交换机怎么配置，PC上wireshark抓到的报文总是没有VLAN tag，是什么情况？

## 原因

一些网卡驱动默认会在接收数据包的时候过滤VLAN TAG，使得用wireshark等软件抓到的报文中不含VLAN TAG。

因此，能否在 Wireshark 中看到 VLAN 标记取决于网卡及其驱动程序对 VLAN 标记的处理方式。

## 解决方式

大多数“简单”网络适配器（例如广泛使用的 Realtek）及其驱动程序将简单地将 VLAN 标记传递给上层以处理这些。在这种情况下，Wireshark 将看到 VLAN 标记并可以处理和显示它们。

一些更复杂的适配器将处理网卡适配器和 / 或驱动程序中的 VLAN 标记，这包括一些 Intel/Broadcom 网卡适配器。



解决方法是修改Windows 注册表，改变网卡的模式，让其捕获报文后不主动去掉VLAN tag：

打开 `compmgmt.msc` --> `设备管理器` --> `网络适配器`,

找到需要抓包的网卡，点击右键 -->`属性` ，再选择`详细信息` ---> `属性` --> `驱动程序关键字`

记录该值，类似于 `{4d36e972-e325-11ce-bfc1-08002be10318}\0002` 的值 。

然后：

- 打开注册表编辑器 regedit
- 找到以下的路径，由于该路径下有很多文件夹，每个文件夹对应PC上的各个网卡: 

`HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\Class\{4d36e972-e325-11ce-bfc1-08002be10318}\0002`

- 新建注册表项（DWORD32类型）



不同厂商对应的注册表键值不相同，例如：

### 1、Mellanox 网卡

来自 [Ethernet Registry Keys - NVIDIA Docs](https://docs.nvidia.com/networking/display/winofv55052000/ethernet+registry+keys)

| **Value Name**   | **Default Value** | **Description** |
| ---------------- | ------------------ | --------------- |
| *PriorityVLANTag | 3 (Packet Priority & VLAN Enabled) | 启用发送和接收 IEEE 802.3ac 标记帧，其中包括：<br />   -  用于优先级标记数据包的 802.1p QoS（服务质量）标记。<br />   -  VLAN 的 802.1Q 标记。<br />启用此功能后，Mellanox 驱动程序支持发送和接收带有 VLAN 和 QoS 标签的数据包。 |
| PromiscuousVlan  | 0                                  | 指定是否启用混杂 VLAN。设置此参数后，所有带有 VLAN 标记的数据包都会传递到上层，而不执行任何过滤。有效值为：<br />       0：禁用<br />       1：启用 |

### 2、intel 网卡

来自 <https://www.intel.com/content/www/us/en/support/articles/000005498/ethernet-products.html> 

要允许标记VLAN的帧传递到您的数据包捕获软件，请添加注册表项 DWORD 类型 和 value，或更改注册表项的值。不同驱动程序对应的注册表DWORD名称不同：

| 适配器驱动程序                         | 注册表项 Registry Key |
| -------------------------------------- | --------------------- |
| e1g, e1e, e1y                          | MonitorModeEnabled    |
| e1c, e1d, e1k, e1q, e1r, ixe, ixn, ixt | MonitorMode           |

创建或更改注册表项 （DWORD 类型）MonitorModeEnabled 时，将 dword 值设置为以下值之一：

- **0** — 禁用（不存储错误数据包、不存储 CRC、去除 802.1Q VLAN 标记）
- **1** — 已启用（存储恶意数据包。存储 CRC。不剥离 802.1Q vlan 标签）

在创建或修改注册表项 （DWORD 类型） MonitorMode 时，将 dword 值设置为以下选项之一：

- **0** — 禁用（不存储错误数据包、不存储 CRC、去除 802.1Q VLAN 标记）
- **1** — 启用（接收错误/残帧/无效的 CRC 数据包。将 CRC 附加到数据包。不要按照正常操作去除 VLAN 标记并忽略发送到其他 VLAN 的数据包。

### 3、Broadcom 网卡

1. 搜索“TxCoalescingTicks”并确保这是您拥有的唯一实例。
2. 右键单击实例编号（例如 0008）并添加一个新的字符串值。
3. 输入“PreserveVlanInfoInRxPacket”并为其赋值“1”。

来自 <https://wiki.wireshark.org/CaptureSetup/VLAN#broadcom> 



重启电脑后再进行抓包，就会发现VLAN tag不会被网卡移除了.

