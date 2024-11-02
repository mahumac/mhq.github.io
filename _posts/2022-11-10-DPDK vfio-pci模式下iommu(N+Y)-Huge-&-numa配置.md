---
title: DPDK:vfio-pci模式下iommu(N+Y)-Huge & numa配置
date: 2022-11-10 +0800
categories: [Linux, DPDK]
description: 
pin: false
tags: [Linux,DPDK,intel] 
---



# DPDK:vfio-pci模式下iommu(N+Y)-Huge & numa配置

# DPDK 安装

```shell
# RHEL/Rocky 8
sudo yum install dpdk dpdk-tools dpdk-doc 
```

# 一、DPDK Hugepages配置：

Hugepages是DPDK用于提升性能的重要手段。 通过使用Hugepages，可以降低内存页数，减少TLB页表数量，增加TLB hit率。

在Linux上设置Hugepages

- 有两种大小：2MB  和 1GB

- 配置大页内存,有两个时机: 

  - **boot time** （推荐）

    boot time的配置写在`/etc/default/grub`里，修改Kernel cmdline传递给内核

  - **run time**

    修改sysfs节点 `/sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages`

### 设置2M内存页

系统一般都支持大页内存，目前大部分都是将2M作为默认配置页，使用默认大页配置，设置巨页
分配巨页1024*2M=2G并查看大页分配数目

```Shell
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cat /proc/meminfo| grep Huge
```

对于2 MB页面，还可以选择在系统启动后进行分配。 这是通过修改 /sys/devices/中的nr_hugepages节点来实现的。 对于单节点系统，若需要1024个页面，可使用如下命令：

```shell
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```


在NUMA机器上，页面的需要明确分配在不同的node上（若只有一个node只需要分配一次）：

```shell
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

这种配置方式的优点是配置简单，无需编译、重启，但是无法配置1GB这样的大hugepages。

### 设置1G内存页（DPDK推荐）

<!--==DPDK官方建议，64位的应用应配置1GB hugepages。==-->

启用1G内存页，一般64位机器均推荐1G大内存页，启用方式如下：

通过修改kernel command line可以在kernel初始化时传入Hugepages相关参数并进行配置。
具体的操作步骤如下：

1. **修改grub文件**

   修改`/etc/default/grub` 文件，在 `GRUB_CMDLINE_LINUX` 中加入如下配置:

   ```shell
   default_hugepagesz=1G hugepagesz=1G hugepages=8
   ```

   该配置表示默认的hugepages的大小为1G，设置的hugepages大小为1G，hugepages的页数为8页，即以8个1G页面的形式保留8G的hugepages内存
   修改后的grub文件示例如下：

   ```shell
   GRUB_TIMEOUT=5
   GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
   GRUB_DEFAULT=saved
   GRUB_DISABLE_SUBMENU=true
   GRUB_TERMINAL_OUTPUT="console"
   GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap default_hugepagesz=1G hugepagesz=1G hugepages=8 rhgb quiet"
   GRUB_DISABLE_RECOVERY="true"
   ```

2. **激活grub配置更改**
   手动编辑 GRUB 2 配置文件后，需要运行 `grub2-mkconfig -o /boot/grub2/grub.cfg` 以激活更改。

3. **重启**

   通过reboot命令重启，随后可以通过`cat /proc/cmdline` 查看kernel的command line是否包含之前的配置。
   也可以通过`cat /proc/meminfo | grep Huge`命令查看是否设置成功，若设置成功可以看到如下配置:

   ```shell
   [root@localhost]~/dpdk-stable-21.11.2/usertools# cat /proc/meminfo | grep Huge
   AnonHugePages:    389120 kB
   ShmemHugePages:        0 kB
   FileHugePages:         0 kB
   HugePages_Total:       8
   HugePages_Free:        8
   HugePages_Rsvd:        0
   HugePages_Surp:        0
   Hugepagesize:    1048576 kB
   Hugetlb:         8388608 kB
   ```

   <!--==DPDK官方建议，64位的应用应配置1GB hugepages。==-->

   这种配置方式的优点是可以在系统开机时即配置预留好hugepages，避免系统运行起来后产生内存碎片；另外，对于较大的例如1G pages，是不支持在系统运行起来后配置的，只能通过kernel cmdline的方式进行配置。

   注：对于双插槽的NUMA系统（dual-socket NUMA system），预留的hugepages会被均分至两个socket，可以通过`lscpu`查看CPU信息：

   ```shell
   [root@localhost]~/dpdk-stable-21.11.2/usertools# lscpu
   Architecture:        x86_64
   CPU op-mode(s):      32-bit, 64-bit
   Byte Order:          Little Endian
   CPU(s):              128
   On-line CPU(s) list: 0-127
   Thread(s) per core:  2
   Core(s) per socket:  32
   Socket(s):           2
   NUMA node(s):        2
   Vendor ID:           GenuineIntel
   BIOS Vendor ID:      Intel
   CPU family:          6
   Model:               106
   Model name:          Intel(R) Xeon(R) Platinum 8375C CPU @ 2.90GHz
   BIOS Model name:     Intel(R) Xeon(R) Platinum 8375C CPU @ 2.90GHz
   ```

   可见测试主机使用的是双socket的NUMA系统。

   

4. **DPDK使用Hugepages**

   预留好Hugepages之后，想要让DPDK使用预留的Hugepages，需要进行挂载操作:

   ```shell
   mkdir /dev/hugepages_1GB
   mount -t hugetlbfs -o pagesize=1G nodev /dev/hugepages_1GB
   ```

   可以将这个挂载点添加到`/etc/fstab`中，这样可以永久生效，即重启后也仍然可以生效:

   ```shell
   # vim /etc/fstab 添加以下内容
   nodev  /dev/hugepages_1GB  hugetlbfs  defaults  0  0
   ```

   对于**1GB pages**，在	`/etc/fstab` 中必须指定默认参数 pagesize=1GB, 否则将使用默认的大小.

   ```shell
   nodev  /dev/hugepages_1GB   hugetlbfs  pagesize=1GB  0  0
   ```

   添加上面这样一行内容至/etc/fstab后并重启，可以通过`df -a | grep huge`看到挂载成功：

   ```shell
   # df -a | grep huge
   
   hugetlbfs               0        0          0     - /dev/hugepages
   nodev                   0        0          0     - /dev/hugepages_1GB
   ```

   

# 二、DPDK驱动VFIO-PCI

===========================DPDK驱动VFIO-PCI==============================
**DPDK 驱动加载有三种方式：**

- `uio`（Userspace I/O）
- `vfio` +` iommu禁用`
- `vfio-pci` +` iommu启用` （**推荐**）

不同的PMD (Poll Mode Driver )需要不同的内核驱动程序才能正常工作。 取决于正在使用的PMD，应加载相应的内核驱动程序并绑定到网络端口。

### VFIO

VFIO 是一个强大而安全的驱动程序，它依赖于 IOMMU 保护。要使用 VFIO，必须加载模块：`vfio-pci`

```shell
sudo modprobe vfio-pci
sudo modprobe vfio-pci enable_sriov=1
```

要使用完整的 VFIO 功能，内核和 BIOS 都必须支持并配置为使用 IO 虚拟化（例如 Intel® VT-d）。



### (一)、UIO（Userspace I/O） 

UIO是运行在用户空间的I/O技术。Linux系统中一般的驱动设备都是运行在内核空间，而在用户空间用应用程序调用即可。

而UIO则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。

在许多情况下，Linux内核中包含的标准`uio_pci_generic`模块可以提供uio功能。 可以使用以下命令加载此模块：

```shell
sudo modprobe uio_pci_generic
```

除了Linux内核中包含的标准`uio_pci_generic`模块，DPDK也提供了一个可替代的igb_uio模块，可以在kmod路径中找到。可以通过以下方法加载igb_uio模块。

```shell
sudo modprobe uio
sudo insmod kmod/igb_uio.ko
```


如果用于DPDK的设备绑定为`uio_pci_generic`内核模块，需要确保IOMMU已禁用或passthrough。 以intel x86_64系统为例，可以在的`GRUB_CMDLINE_LINUX`中添加`intel_iommu = off`或`intel_iommu = on iommu = pt`。

### (二)、 启用vfio-pci、iommu禁用

VFIO no-IOMMU mode

如果系统上没有可用的 IOMMU，VFIO 仍然可以使用，但必须加载一个额外的模块参数：

```shell
modprobe vfio enable_unsafe_noiommu_mode=1 
```

或者

```shell
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode 
```

之后，VFIO 就可以像往常一样与硬件设备一起使用了。

**警告：**由于 no-IOMMU 模式放弃了 IOMMU 保护，因此它本质上是不安全的。也就是说，在 IOMMU 不可用的情况下，它确实可以让用户保持 VFIO 拥有的设备访问和编程程度。

PS：以挂载ens7 网卡，pci 0000:02:05.0 为例

  1. 关闭计划用dpdk接管的网卡接口，并查询其pci端口号，可以通过`lspci |grep Ethernet`查看。此时需要确认本机物理网卡或虚拟网卡为DPDK支持类型，查询网址https://core.dpdk.org/supported/

     ```shell
      ifconfig ens7 down
      # ip link set ens7 down
     ```

2. 安装NIC网卡驱动模块并启动非安全NOIOMMU模式

   ```shell
   modprobe vfio-pci
   echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode 
   ```

**注意事项：**

如果，发现dpdk 没有挂载上网卡那请手动绑定一下，手动执行在第2步之后

```shell
./usertools/dpdk-devbind.py --bind=vfio-pci 0000:02:05.0
```


### (三)、VFIO: 启用vfio-pci、iommu启用 （推荐）

l2fwd运行绑定网卡方式：（启用vfio-pci、iommu启用）VFIO（推荐）

VFIO与UIO相比，它更加强大和安全，依赖于IOMMU。 要使用VFIO，需要:

- Linix kernel version>=3.6.0

- 内核和BIOS必须支持并配置为使用`IO virtualization`（例如Intel®VT-d）

在确认硬件配置支持的情况下，要使用VFIO驱动绑定到NIC必须先使能iommu，否则会导致绑定失败。具体的现象就是查看或修改sysfs节点`/sys/bus/pci/drivers/vfio-pci/bind`出现io错误，以及dmesg中出现：

```shell
# dmesg | grep iommu
vfio-pci: probe of 0000:05:00.0 failed with error -22
```

使能iommu的方法也是修改kernel的command line将`iommu=pt  intel_iommu=on`传入，具体步骤：

1. **修改grub文件**
   修改`/etc/default/grub`文件，在`GRUB_CMDLINE_LINUX`中加入如下配置:

   ```shell
   # 在GRUB_CMDLINE_LINUX 中添加 iommu=pt intel_iommu=on
   
   GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/cs-swap rd.lvm.lv=cs/root rd.lvm.lv=cs/swap rhgb quiet iommu=pt intel_iommu=on default_hugepagesz=1G hugepagesz=1G hugepages=8"
   ```

2. **激活grub配置更改**
   手动编辑 GRUB 2 配置文件后，需要运行 `grub2-mkconfig -o /boot/grub2/grub.cfg` 以激活更改。

3. **重启**
   通过reboot命令重启，随后可以通过`cat /proc/cmdline`查看kernel的command line是否包含之前的配置。

iommu配置成功后，dmesg中会有iommu配置group的log，可以通过`dmesg | grep iommu`查看：

```shell
[    1.784044] pci 0000:00:00.0: Adding to iommu group 0
[    1.784100] pci 0000:00:04.0: Adding to iommu group 1
[    1.784152] pci 0000:00:04.1: Adding to iommu group 2
[    1.784202] pci 0000:00:04.2: Adding to iommu group 3
... ...
```

即表示iommu使能成功。
随后需要调用modprobe来加载VFIO的驱动:

```shell
sudo modprobe vfio-pci
# 使其开机自动加载
echo "vfio-pci" > /etc/modules-load.d/vfio_pci.conf 
```

如果系统上没有可用的 IOMMU，VFIO 仍然可以使用，但必须加载一个额外的模块参数：

```shell
modprobe vfio enable_unsafe_noiommu_mode=1 
```

或者

```shell
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode 
```

之后，VFIO 就可以像往常一样与硬件设备一起使用了。

**驱动绑定NIC**

上述的UIO和VFIO驱动可以加载一项，也可以全都加载。但是在驱动绑定NIC的时候，只能选择一种驱动绑定到NIC，这里采用VFIO驱动。

可以调用dpdk路径下的`usertools/dpdk-devbind.py`实用脚本来进行VFIO驱动与NIC绑定，需要注意的是使用这个脚本进行绑定（bind）动作时是需要root权限的。
可以调用脚本传入 --status 查看当前的网络端口的状态：

```shell
[root@localhost]~/dpdk-stable-21.11.2/usertools# ./dpdk-devbind.py --status

Network devices using kernel driver
===================================
0000:01:00.0 'NetXtreme BCM5720 Gigabit Ethernet PCIe 165f' if=eno3 drv=tg3 unused= 
0000:01:00.1 'NetXtreme BCM5720 Gigabit Ethernet PCIe 165f' if=eno4 drv=tg3 unused= 
0000:18:00.0 'BCM57416 NetXtreme-E Dual-Media 10G RDMA Ethernet Controller 16d8' if=eno1np0 drv=bnxt_en unused= 
0000:18:00.1 'BCM57416 NetXtreme-E Dual-Media 10G RDMA Ethernet Controller 16d8' if=eno2np1 drv=bnxt_en unused= *Active*
0000:3b:00.0 'Ethernet Controller E810-C for QSFP 1592' if=ens1f0 drv=ice unused= 
0000:3b:00.1 'Ethernet Controller E810-C for QSFP 1592' if=ens1f1 drv=ice unused= 
0000:af:00.0 'Ethernet Controller E810-C for QSFP 1592' if=ens4f0 drv=ice unused= 
0000:af:00.1 'Ethernet Controller E810-C for QSFP 1592' if=ens4f1 drv=ice unused= 
```

可以看到，当前 Intel E810-C 网卡的状态都是**Network devices using kernel driver**，使用的是kernel的 **ice** 驱动  (**drv=ice**) 。
随后可以调用脚本传入--bind将网卡 0000:3b:00.0 、0000:3b:00.1，也就是ens1f0、ens1f1绑定到VFIO驱动：

```shell
./usertools/dpdk-devbind.py --bind=vfio-pci 0000:3b:00.0 0000:3b:00.1
```

再次调用脚本传入`dpdk-devbind.py --status`，可以看到

```shell
[root@localhost]~/dpdk-stable-21.11.2/usertools# ./dpdk-devbind.py --status

Network devices using DPDK-compatible driver
============================================
0000:3b:00.0 'Ethernet Controller E810-C for QSFP 1592' drv=vfio-pci unused=ice
0000:3b:00.1 'Ethernet Controller E810-C for QSFP 1592' drv=vfio-pci unused=ice

Network devices using kernel driver
===================================
0000:01:00.0 'NetXtreme BCM5720 Gigabit Ethernet PCIe 165f' if=eno3 drv=tg3 unused=vfio-pci 
0000:01:00.1 'NetXtreme BCM5720 Gigabit Ethernet PCIe 165f' if=eno4 drv=tg3 unused=vfio-pci 
0000:18:00.0 'BCM57416 NetXtreme-E Dual-Media 10G RDMA Ethernet Controller 16d8' if=eno1np0 drv=bnxt_en unused=vfio-pci 
0000:18:00.1 'BCM57416 NetXtreme-E Dual-Media 10G RDMA Ethernet Controller 16d8' if=eno2np1 drv=bnxt_en unused=vfio-pci *Active*
0000:af:00.0 'Ethernet Controller E810-C for QSFP 1592' if=ens4f0 drv=ice unused=vfio-pci 
0000:af:00.1 'Ethernet Controller E810-C for QSFP 1592' if=ens4f1 drv=ice unused=vfio-pci 
```


设备 0000:3b:00.0 、0000:3b:00.1 的状态变为 **Network devices using DPDK-compatible driver**，使用的驱动为`drv=vfio-pci`，若想要恢复为kernel默认的igb驱动，则可以继续调用脚本：

```shell
./usertools/dpdk-devbind.py --bind=igb 0000:3b:00.0  0000:3b:00.1
```

==**注意事项：**==

**要使`dpdk-devbind.py --bind`绑定永久生效，可以使用 `driverctl` 实用程序：**

```shell
# 安装 driverctl 实用程序：yum install driverctl 或 apt install driverctl

driverctl -v set-override 0000:3b:00.0 vfio-pci
driverctl -v set-override 0000:3b:00.1 vfio-pci
```

使用`driverctl list-overrides`，查看是否生效

```shell
[root@localhost]# driverctl list-overrides
0000:3b:00.0 vfio-pci
0000:3b:00.1 vfio-pci
```

使用 `driverctl  unset-override`参数可以解除绑定

# 后记

基于DPDK的简单应用编译与运行方法可以查看：

==================启用numa编译选项===============================
又踩了一个坑，在编译启用numa时候，若直接编译，dpdk是启用不了numa的，需要安装一些基础库

yum install numactl
yum install numactl-libs
yum install numactl-devel
需要安装完上面这些库，才能编译出带numa的dpdk库

numa需要加深认识，留待以后详细了解，较好的文档如下：

[译文：ovs+dpdk中的“vHost User NUMA感知”特性 - 张浮生 - 博客园](https://www.cnblogs.com/neooelric/p/7090097.html)

[9.3. 调度 NUMA 感知工作负载 | Red Hat Product Documentation](https://docs.redhat.com/zh_hans/documentation/openshift_container_platform/4.14/html/scalability_and_performance/cnf-scheduling-numa-aware-workloads-overview_numa-aware)

[使用带有 Open vSwitch 的DPDK创建vHost 用户非一致性内存访问感知](https://www.intel.cn/content/www/cn/zh/developer/articles/technical/vhost-user-numa-awareness-in-open-vswitch-with-dpdk.html)

---------

# 如何在 Intel 平台上使用 NIC 获得最佳性能

## 硬件和内存要求

为获得最佳性能，请使用 Intel Xeon 级服务器系统，例如 Ivy Bridge、Haswell 或更新版本。

确保每个内存通道至少插入一个内存 DIMM，并且每个内存通道的内存大小至少为 4GB。 **注意**：这是对性能最直接的影响之一。

您可以使用以下方法检查内存配置`dmidecode`：

```shell
# dmidecode -t memory | grep Locator

Locator: DIMM_A1
Bank Locator: NODE 1
Locator: DIMM_A2
Bank Locator: NODE 1
Locator: DIMM_B1
Bank Locator: NODE 1
Locator: DIMM_B2
Bank Locator: NODE 1
...
Locator: DIMM_G1
Bank Locator: NODE 2
Locator: DIMM_G2
Bank Locator: NODE 2
Locator: DIMM_H1
Bank Locator: NODE 2
Locator: DIMM_H2
Bank Locator: NODE 2
```

上面的示例输出显示总共 8 个通道，从`A`到`H`，其中每个通道有 2 个 DIMM。

您还可以使用`dmidecode`来确定内存频率：

```shell
# dmidecode -t memory | grep Speed

Speed: 2133 MHz
Configured Clock Speed: 2134 MHz
Speed: Unknown
Configured Clock Speed: Unknown
Speed: 2133 MHz
Configured Clock Speed: 2134 MHz
Speed: Unknown
...
Speed: 2133 MHz
Configured Clock Speed: 2134 MHz
Speed: Unknown
Configured Clock Speed: Unknown
Speed: 2133 MHz
Configured Clock Speed: 2134 MHz
Speed: Unknown
Configured Clock Speed: Unknown
```

输出显示速度为 2133 MHz (DDR4) 和未知（不存在）。这与之前的输出一致，显示每个通道都有一个内存条。

## 网络接口卡要求

使用[DPDK 支持](https://core.dpdk.org/supported/)的高端 NIC，例如 Intel XL710 40GbE。

确保每个 NIC 都刷入了最新版本的 NVM/固件。

使用 PCIe Gen3 插槽，例如 Gen3`x8`或 Gen3 `x16`，因为 PCIe Gen2 插槽无法为 2 x 10GbE 及以上提供足够的带宽。您可以使用`lspci`类似以下的方法来检查 PCI 插槽的速度：

```shell
lspci -s 03:00.1 -vv | grep LnkSta

LnkSta: Speed 8GT/s, Width x8, TrErr- Train- SlotClk+ DLActive- ...
LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ ...
```

将 NIC 插入 PCI 插槽时，请始终检查标题，例如 CPU0 或 CPU1 以指示它连接到哪个插槽。

应注意 NUMA。如果您使用来自不同 NIC 的 2 个或更多端口，最好确保这些 NIC 在同一个 CPU 插槽上。下面进一步显示了如何确定这一点的示例。

## BIOS 设置

以下是有关 BIOS 设置的一些建议。不同平台会有不同的BIOS命名，以下主要供参考：

1. 为系统建立稳定状态，考虑检查最佳性能特征所需的 BIOS 设置，例如优化性能或能效。
2. 使 BIOS 设置与您正在测试的应用程序的需要相匹配。
3. 通常，**性能**作为 CPU 功率和性能策略是一个合理的起点。
4. 考虑使用 Turbo Boost 来增加核心频率。
5. 测试网卡物理功能时禁用所有虚拟化选项，如果要使用VFIO则开启VT-d。

## Linux 引导选项

以下是有关 GRUB 引导设置的一些建议：

1. 修改grub文件。

2. 通过 grub 配置预留 1G 大页面。例如保留 8 个 1G 大小的大页面：

   ```shell
   default_hugepagesz=1G hugepagesz=1G hugepages=8
   ```

3. 隔离将用于 DPDK 的 CPU 内核。例如：

   ```shell
   isolcpus=2,3,4,5,6,7,8
   ```

4. 如果它想使用 VFIO，请使用以下额外的 grub 参数：

   ```shell
   iommu=pt intel_iommu=on
   ```

==**激活grub配置更改**==
手动编辑 GRUB 2 配置文件后，需要运行 `grub2-mkconfig -o /boot/grub2/grub.cfg` 以激活更改。

##  运行DPDK前的配置

1. 保留大页面。有关更多详细信息，请参阅前面关于[在 Linux 环境中使用大页面的部分。](https://doc.dpdk.org/guides/linux_gsg/sys_reqs.html#linux-gsg-hugepages)

   ```shell
   # Get the hugepage size.
   awk '/Hugepagesize/ {print $2}' /proc/meminfo
   
   # Get the total huge page numbers.
   awk '/HugePages_Total/ {print $2} ' /proc/meminfo
   
   # Unmount the hugepages.
   umount `awk '/hugetlbfs/ {print $2}' /proc/mounts`
   
   # Create the hugepage mount folder.
   mkdir -p /mnt/huge
   
   # Mount to the specific folder.
   mount -t hugetlbfs nodev /mnt/huge
   ```

2. 使用 DPDK`cpu_layout`实用程序检查 CPU 布局：

   ```shell
   cd dpdk_folder
   usertools/cpu_layout.py
   ```

   或者运行`lscpu`检查每个插槽上的内核。

3. 检查您的 NIC id 和相关的套接字 id：

   ```shell
   # List all the NICs with PCI address and device IDs.
   lspci -nn | grep Eth
   ```

   例如，假设您的输出如下：

   ```shell
   82:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
   82:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
   85:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
   85:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
   ```

   检查 PCI 设备相关的 numa node id：

   ```
   cat /sys/bus/pci/devices/0000\:xx\:00.x/numa_node
   ```

   通常`0x:00.x`在插槽 0 和`8x:00.x`插槽 1 上。 **注意**：要获得最佳性能，请确保核心和 NIC 在同一插槽中。在上面的例子`85:00.0`中是在插槽 1 上，应该由插槽 1 上的内核使用以获得最佳性能。

\4. 查看需要加载哪些内核驱动，是否需要解除网口与内核驱动的绑定。有关 DPDK 设置和 Linux 内核要求的更多详细信息，请参阅从源代码和[Linux 驱动程序](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html#linux-gsg-linux-drivers)[编译 DPDK 目标](https://doc.dpdk.org/guides/linux_gsg/build_dpdk.html#linux-gsg-compiling-dpdk)。

参考文档：[11. How to get best performance with NICs on Intel platforms — Data Plane Development Kit 24.11.0-rc1 documentation](https://doc.dpdk.org/guides/linux_gsg/nic_perf_intel_platform.html)