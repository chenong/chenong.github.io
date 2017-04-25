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

![DPDK l2fwd run error log](https://github.com/chenong/chenong.github.io/blob/master/image/dpdk_bind_i210_pci_error/dpdk_l2fwd_run_error_log.jpg "DPDK l2fwd run error log")

网卡的绑定情况，如下图所示：

![DPDK pci bind status](https://github.com/chenong/chenong.github.io/blob/master/image/dpdk_bind_i210_pci_error/dpdk_bind_i210_pci_status_info.jpg "DPDK pci bind status")

网卡的PCI信息，如下图所示：

![DPDK pci info](https://github.com/chenong/chenong.github.io/blob/master/image/dpdk_bind_i210_pci_error/dpdk_inter_i210_pci_info.jpg "DPDK pci info")

DPDK 版本： dpdk-17.02

初步判断DPDK环境以及l2fwd运行参数等信息都没有问题。

最后通过调试DPDK网卡驱动与PCI的关联过程得知，DPDK实现了I210网卡的用户态驱动，但是对于0x157B子型号的I210网卡并没有注册，导致网卡和驱动关联不上。

修改dpdk-17.02/drivers/net/e1000/igb_ethdev.c中static const struct rte_pci_id pci_id_igb_map[]全局结构注册该I210网卡，如下图所示：

![dpdk_i210_supports](https://github.com/chenong/chenong.github.io/blob/master/image/dpdk_bind_i210_pci_error/dpdk_i210_supports.jpg "dpdk_i210_supports")

重新编译DPDK以及l2fwd，

运行后收发包正常。
