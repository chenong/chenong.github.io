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

&emsp;&emsp;今天在测试DPDK l2fwd时通过DPDK脚本绑定了设备上面的网卡，但是在运行l2fwd时，出错显示未检测到有可用的网卡程序退出。
