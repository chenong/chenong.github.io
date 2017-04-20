---
layout: post
title:  "TLDK l4fwd usage"
date:   2017-04-19 10:24:36
categories: TLDK
tags: TLDK l4fwd DPDK
---

* content
{:toc}


TLDK l4fwd 使用用例

##1. 简介

&emsp;&emsp;l4fwd是演示和测试TLDK TCP/UDP功能的示例应用程序。根据用户配置，可以实现简单的send、recv或者send+recv已经打开的TCP/UDP数据流。他还实现了TCP/UDP不同流之间的数据包转发功能。所以可以使用l4fwd应用程序作为简单的TCP/UDP代理。

&emsp;&emsp;l4fwd应用程序逻辑上分为两个部分：后端(BE) 和 前端(FE)。

###1.1 后端 (BE)

###1.2 前端 (FE)
