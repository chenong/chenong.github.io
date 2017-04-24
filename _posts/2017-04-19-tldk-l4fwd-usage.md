---
layout: post
title:  "TLDK l4fwd usage"
date:   2017-04-19 10:24:36
categories: TLDK
tags: TLDK l4fwd DPDK
---

* content
{:toc}

# TLDK l4fwd 使用用例

## 1. 简介

&emsp;&emsp;l4fwd是演示和测试TLDK TCP/UDP功能的示例应用程序。根据用户配置，可以实现简单的send、recv或者send+recv已经打开的TCP/UDP数据流。他还实现了TCP/UDP不同流之间的数据包转发功能。所以可以使用l4fwd应用程序作为简单的TCP/UDP代理。

&emsp;&emsp;l4fwd应用程序逻辑上分为两个部分：后端(BE) 和 前端(FE)。

### 1.1 后端 (BE)

后端负责:

- RX over DPDK ports and feed them into TCP/UDP TLDK context(s) (via tle_*_rx_bulk). 通过DPDK网卡接收报文并将其注入到TLDK TCP/UDP上下文中.

- retrieve packets ready to be send out from TCP/UDP TLDK context(s) and TX them over destined DPDK port. 检索TLDK TCP/UDP上下文中是否有需要发送的报文,并通过DPDK网卡发送出去.

- Multiple RX/TX queues per port are supported by RSS. Right now the number of TX is same as the number of RX queue. 通过RSS支持每个网卡的多RX/TX队列. 现在TX队列的数量等于RX队列的数量.

Each BE lcore can serve multiple DPDK ports, TLDK TCP/UDP contexts. 每一个BE locre能过服务多个DPDK网卡, TLDK上下文.

    BE configuration record format:
    
    port=<uint>,addr=<ipv4/ipv6>,masklen=<uint>,mac=<ether><mtu>
    
    port -    DPDK port id to be used to send packets to the destination. It is an mandatory option.
    addr -    destination network address. It is an mandatory option.
    masklen - destination network prefix length. It is an mandatory option.
    mac -     destination Ethernet address. It is an mandatory option.
    mtu -     MTU to be used on that port (= application data size + L2/L3/L4 headers sizes, default=1514). It is an optional option.

    Below are some example of BE entries

    port=0,masklen=16,addr=192.168.0.0,mac=01:de:ad:be:ef:01
    port=0,addr=2001:4860:b002::,masklen=64,mac=01:de:ad:be:ef:01

These examples are also available in be.cfg file.

```cfg
#
# l4fwd BE config file example
#
port=0,masklen=16,addr=192.168.0.0,mac=01:de:ad:be:ef:01
port=0,addr=2001:4860:b002::,masklen=64,mac=01:de:ad:be:ef:01
```

### 1.2 前端 (FE)

前端负责:
- to open configured TCP/UDP streams and perform send/recv over them. These streams can belong to different TCP/UDP contexts. 执行发送/接收配置的TCP/UDP流。 这些流可以属于不同的TCP/UDP上下文。

Each lcore can act as BE and/or FE. 每个lcore可以作为 BE 和/或 FE。

In UDP mode the application can reassemble input fragmented IP packets, and fragment outgoing IP packets (if destination MTU is less then packet size). 在UDP模式下，应用程序可以重新组合输入分片的IP数据包，并分片出局IP数据包（如果目标MTU小于数据包大小）。

    FE configuration record format:
    
    lcore=<uint>,op=<"rx|tx|echo|fwd">,laddr=<ip>,lport=<uint16>,raddr=<ip>,rport=<uint16>, [txlen=<uint>,fwladdr=<ip>,fwlport=<uint16>,fwraddr=<ip>,fwrport=<uint16>,belcore=<uint>]
    
    lcore -   EAL lcore to manage that stream(s) in the FE. It is an mandatory option.
    belcore - EAL lcore to manage that stream(s) in the BE. It is an optional option. lcore and belcore can specify the same cpu core. lcore和belcore可以指定相同的cpu内核。
    op -      operation to perform on that stream: 在该流上执行的操作
              "rx" - do receive only on that stream.
              "tx" - do send only on that stream.
              "echo" - mimic 模仿recvfrom(..., &addr);sendto(..., &addr); on that stream.
              "fwd" - forward packets between streams.
              It is an mandatory option.
    laddr -   local address for the stream to open. It is an mandatory option.
    lport -   local port for the stream to open. It is an mandatory option.
    raddr -   remote address for the stream to open. It is an mandatory option.
    rport -   remote port for the stream to open. It is an mandatory option.
    txlen -   data length sending in each packet (mandatory for "tx" mode only).
    fwladdr - local address for the forwarding stream(s) to open (mandatory for "fwd" mode only).
    fwlport - local port for the forwarding stream(s) to open (mandatory for "fwd" mode only).
    fwraddr - remote address for the forwarding stream(s) to open (mandatory for "fwd" mode only).
    fwrport - remote port for the forwarding stream(s) to open (mandatory for "fwd" mode only).

    Below are some example of FE entries

    lcore=3,op=echo,laddr=192.168.1.233,lport=0x8000,raddr=0.0.0.0,rport=0

    lcore=3,op=tx,laddr=192.168.1.233,lport=0x8001,raddr=192.168.1.56,rport=0x200,txlen=72

    lcore=3,op=rx,laddr=::,lport=0x200,raddr=::,rport=0,txlen=72

    lcore=3,op=fwd,laddr=0.0.0.0,lport=11211,raddr=0.0.0.0,rport=0,fwladdr=::,fwlport=0,fwraddr=2001:4860:b002::56,fwrport=11211


These examples are also available in fe.cfg file with some more explanation.

```cfg
#
# udpfwd FE config file example
#

# open IPv4 stream with local_addr=192.168.1.233:32768,
# and remote_addr as wildcard (any remote addressi/port allowed).
# use it echo mode - for any received packet - send it back to the source
lcore=3,op=echo,laddr=192.168.1.233,lport=0x8000,raddr=0.0.0.0,rport=0

# open IPv4 stream with specified local/remote address/port and
# do send only over that stream.
lcore=3,op=tx,laddr=192.168.1.233,lport=0x8001,raddr=192.168.1.56,rport=0x200,txlen=72

# open IPv6 stream with specified local port (512) probably over multiple
# eth ports, and do recv only over that stream.
lcore=3,op=rx,laddr=::,lport=0x200,raddr=::,rport=0,txlen=72

# fwd mode example.
# open IPv4 stream on local port 11211 (memcached) over all possible ports.
# for each new flow, sort of tunnel will be created, i.e:
# new stream will be opend to communcate with forwarding remote address,
# so all packets with <laddr=A:11211,raddr=X:N> will be forwarded to
# <laddr=[B]:M,raddr=[2001:4860:b002::56]:11211> and visa-versa.
lcore=3,op=fwd,laddr=0.0.0.0,lport=11211,raddr=0.0.0.0,rport=0,fwladdr=::,fwlport=0,fwraddr=2001:4860:b002::56,fwrport=11211
```
### 1.3 配置文件格式
1111
