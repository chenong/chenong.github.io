---
layout: post
title:  "DPDK bind inter i210 pci error"
date:   2017-04-25 14:57:36
categories: DPDK
tags: DPDK inter i210 port rte_eth_dev_count return 0
---

* content
{:toc}

# DPDK 绑定I210网卡错误

&emsp;&emsp;今天在测试DPDK l2fwd时遇到rte_eth_dev_count() return 0的情况，在博客中做个记录，如下图所示：

```shell
[root@localhost l2fwd]# ./build/l2fwd  -c0xf -n4 -- -p0x3 -q 1 -T 2
EAL: Detected 4 lcore(s)
EAL: Probing VFIO support...
PMD: bnxt_rte_pmd_init() called for (null)
EAL: PCI device 0000:01:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:02:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:03:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:06:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:07:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:08:00.0 on NUMA socket -1
EAL:   probe driver: 8086:157b rte_igb_pmd
EAL: PCI device 0000:0d:00.0 on NUMA socket -1
EAL:   probe driver: 8086:1536 rte_igb_pmd
EAL: PCI device 0000:0e:00.0 on NUMA socket -1
EAL:   probe driver: 8086:1536 rte_igb_pmd
MAC updating enabled
EAL: Error - exiting with code: 1
  Cause: No Ethernet ports - bye
[root@localhost l2fwd]#
```

网卡的绑定情况，如下图所示：

```shell
[root@localhost usertools]# ./dpdk-devbind.py  --status

Network devices using DPDK-compatible driver
============================================
0000:02:00.0 'I210 Gigabit Network Connection' drv=igb_uio unused=
0000:03:00.0 'I210 Gigabit Network Connection' drv=igb_uio unused=

Network devices using kernel driver
===================================
0000:01:00.0 'I210 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*
0000:06:00.0 'I210 Gigabit Network Connection' if=eth3 drv=igb unused=igb_uio *Active*
0000:07:00.0 'I210 Gigabit Network Connection' if=eth4 drv=igb unused=igb_uio
0000:08:00.0 'I210 Gigabit Network Connection' if=eth5 drv=igb unused=igb_uio
0000:0d:00.0 'I210 Gigabit Fiber Network Connection' if=eth6 drv=igb unused=igb_uio
0000:0e:00.0 'I210 Gigabit Fiber Network Connection' if=eth7 drv=igb unused=igb_uio

Other network devices
=====================
<none>

Crypto devices using DPDK-compatible driver
===========================================
<none>

Crypto devices using kernel driver
==================================
<none>

Other crypto devices
====================
<none>
[root@localhost usertools]#
```

网卡的PCI信息，如下图所示：

```shell
[root@localhost l2fwd]# lspci -nn |grep Eth |grep 157b
01:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
02:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
03:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
06:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
07:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
08:00.0 Ethernet controller [0200]: Intel Corporation I210 Gigabit Network Connection [8086:157b] (rev 03)
[root@localhost l2fwd]#
```

DPDK 版本： dpdk-17.02

初步判断DPDK环境以及l2fwd运行参数等信息都没有问题。

最后通过调试DPDK网卡驱动与PCI的关联过程得知，DPDK实现了I210网卡的用户态驱动，但是对于0x157B子型号的I210网卡并没有注册，导致网卡和驱动关联不上。

修改dpdk-17.02/drivers/net/e1000/igb_ethdev.c中static const struct rte_pci_id pci_id_igb_map[]全局结构注册该I210网卡，如下图所示：

```c
/*
 * The set of PCI devices this driver supports
 */
static const struct rte_pci_id pci_id_igb_map[] = {
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_FIBER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_QUAD_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_QUAD_COPPER_ET2) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_NS) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_NS_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82576_SERDES_QUAD) },

        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82575EB_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82575EB_FIBER_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82575GB_QUAD_COPPER) },

        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_FIBER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_SGMII) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_COPPER_DUAL) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_82580_QUAD_FIBER) },

        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I350_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I350_FIBER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I350_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I350_SGMII) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I350_DA4) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_COPPER_FLASHLESS) }, /* add by chenchong */
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_COPPER_OEM1) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_COPPER_IT) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_FIBER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I210_SGMII) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I211_COPPER) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I354_BACKPLANE_1GBPS) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I354_SGMII) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_I354_BACKPLANE_2_5GBPS) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_DH89XXCC_SGMII) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_DH89XXCC_SERDES) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_DH89XXCC_BACKPLANE) },
        { RTE_PCI_DEVICE(E1000_INTEL_VENDOR_ID, E1000_DEV_ID_DH89XXCC_SFP) },
        { .vendor_id = 0, /* sentinel */ },
};
```

E1000_DEV_ID_I210_COPPER_FLASHLESS 宏定义在e1000_hw.h中有定义, 与网卡的device_id相同。

```c
#define E1000_DEV_ID_I210_COPPER_FLASHLESS      0x157B
```

修改完成后，重新编译DPDK以及l2fwd。

测试运行l2fwd收发包正常。
