---
layout: post
title:  "DPDK pktgen usage"
date:   2017-04-24 14:01:36
categories: DPDK
tags: DPDK pktgen
---

* content
{:toc}

# DPDK pktgen 使用用例

## 1. 简介

pktgen(packet gen-erator)该软件基于DPDK快速报文处里框架开发的一个发包工具。
&emsp;

Pktgen提供的功能如下：
1. 能够提供64byte小报的10Gbit线速发包。
2. 能够作为线速的收报机或者发报机。
3. 能够运行时配置、开始或停止发报。
4. 能够显示多个网卡的实时状态。
5. 能够通过迭代源或目标MAC，IP或端口来顺序生成数据包。
6. 能够处理UDP，TCP，ARP，ICMP，GRE，MPLS和Queue-in-Queue的报文。
7. 能够通过TCP连接远程控制。
8. 能够通过Lua进行配置，并可以运行命令脚本来设置可重复的测试用例。
9. 该软件根据BSD许可证完全可用。
&emsp;

## 2. 运行参数

&emsp;&emsp;pktgen像其他DPDK应用程序一样，将命令行参数分解为DPDK环境抽象层（EAL）的参数和应用程序本身的参数。 两组参数使用 -- 的标准约定分开。
&emsp;

### 2.1 EAL运行参数
&emsp;&emsp;The full list of EAL arguments are:

```shell
EAL options:
    -c COREMASK         : A hexadecimal bitmask of cores to run on
    -n NUM              : Number of memory channels
    -v                  : Display version information on startup
    -d LIB.so           : Add driver (can be used multiple times)
    -m MB               : Memory to allocate (see also --socket-mem)
    -r NUM              : Force number of memory ranks (don't detect)
    --xen-dom0          : Support application running on Xen Domain0 without
                          hugetlbfs
    --syslog            : Set syslog facility
    --socket-mem        : Memory to allocate on specific
                          sockets (use comma separated values)
    --huge-dir          : Directory where hugetlbfs is mounted
    --proc-type         : Type of this process
    --file-prefix       : Prefix for hugepage filenames
    --pci-blacklist, -b : Add a PCI device in black list.
                          Prevent EAL from using this PCI device. The argument
                          format is <domain:bus:devid.func>.
    --pci-whitelist, -w : Add a PCI device in white list.
                          Only use the specified PCI devices. The argument
                          format is <[domain:]bus:devid.func>. This option
                          can be present several times (once per device).
                          NOTE: PCI whitelist cannot be used with -b option
    --vdev              : Add a virtual device.
                          The argument format is <driver><id>[,key=val,...]
                          (ex: --vdev=eth_pcap0,iface=eth2).
    --vmware-tsc-map    : Use VMware TSC map instead of native RDTSC
    --base-virtaddr     : Specify base virtual address
    --vfio-intr         : Specify desired interrupt mode for VFIO
                          (legacy|msi|msix)
    --create-uio-dev    : Create /dev/uioX (usually done by hotplug)

EAL options for DEBUG use only:
    --no-huge           : Use malloc instead of hugetlbfs
    --no-pci            : Disable pci
    --no-hpet           : Disable hpet
    --no-shconf         : No shared config (mmap'd files)
```
&emsp;
&emsp;&emsp;The -c COREMASK and -n NUM arguments are required. The other arguments are optional.

&emsp;
&emsp;&emsp;Pktgen需要2个逻辑内核（lcore）才能运行。 第一个lcore，0用于pktgen命令行，用于定时器和在终端上显示运行时实时状态。 附加的1-n被用于执行数据包的接收和发送以及与数据包相关的任何事物。不需要在实际的系统lcore 0上启动。应用程序将使用coremask位图中的第一个lcore作为0核。
&emsp;

### 2.2 pktgen运行参数
  
&emsp;&emsp;The Pktgen commandline usage is:
  
```shell
    ./app/app/``$(target}``/pktgen [EAL options] -- \
                             [-h] [-P] [-G] [-T] [-f cmd_file] \
                             [-l log_file] [-s P:PCAP_file] [-m <string>]
```
```shell
The pktgen arguments are:
Usage:
    -h           Display the help information
    -s P:file    PCAP packet stream file, 'P' is the port number
    -f filename  Command file (.pkt) to execute or a Lua script (.lua) file
    -l filename  Write log to filename
    -P           Enable PROMISCUOUS mode on all ports
    -G           Enable socket support using default server values localhost:0x5606
    -g address   Optional IP address and port number default is (localhost:0x5606)
                 If -g is used that enable socket support as a server application
    -N           Enable NUMA support
    -T           Enable the color output
    -m <string>  Matrix for mapping ports to logical cores
```

&emsp;&emsp;Where the options are:

    -h:  Display the usage/help information shown above:
    
         lspci | grep Ethernet
         This shows a list of all ports in the system. Some ports may not be usable by DPDK/Pktgen. 
         The first port listed is bit 0 or least  signification bit in the -c EAL coremask. Another 
         method is to compile and run the DPDK sample application testpmd to list out the ports DPDK
         is able to use: ./test_pmd -c 0x3 -n 2
         
    -s   P:file: The PCAP packet file to stream. P is the port number.
    
    -f   filename: The script command file (.pkt) to execute or a Lua script (.lua) file. See Running Script Files.

    -l   filename: The filename to write a log to.
    
    -P:  Enable PROMISCUOUS mode on all ports.
    
    -G:  Enable socket support using default server values of localhost:0x5606. See Socket Support for Pktgen.
    
    -g   address: Same as -G but with an optional IP address and port number. See Socket Support for Pktgen.
    
    -T:  Enable color terminal output in VT100
    
    -N:  Enable NUMA support.
    
    -m   <string>: Matrix for mapping ports to logical cores. The format of the port mapping string is defined 
         with a BNF-like grammar as follows:
         BNF: (or kind of BNF)
         <matrix-string>   := """ <lcore-port> { "," <lcore-port>} """
         <lcore-port>      := <lcore-list> "." <port-list>
         <lcore-list>      := "[" <rx-list> ":" <tx-list> "]"
         <port-list>       := "[" <rx-list> ":" <tx-list>"]"
         <rx-list>         := <num> { "/" (<num> | <list>) }
         <tx-list>         := <num> { "/" (<num> | <list>) }
         <list>            := <num> { "/" (<range> | <list>) }
         <range>           := <num> "-" <num> { "/" <range> }
         <num>             := <digit>+
         <digit>           := 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9

```
For example:

1.0, 2.1, 3.2                 - core 1 handles port 0 rx/tx,
                                core 2 handles port 1 rx/tx
                                core 3 handles port 2 rx/tx
1.[0-2], 2.3, ...             - core 1 handle ports 0,1,2 rx/tx,
                                core 2 handle port 3 rx/tx
[0-1].0, [2/4-5].1, ...       - cores 0-1 handle port 0 rx/tx,
                                cores 2,4,5 handle port 1 rx/tx
[1:2].0, [4:6].1, ...         - core 1 handles port 0 rx,
                                core 2 handles port 0 tx,
[1:2].[0-1], [4:6].[2/3], ... - core 1 handles port 0 & 1 rx,
                                core 2 handles port  0 & 1 tx
[1:2-3].0, [4:5-6].1, ...     - core 1 handles port 0 rx, cores 2,3 handle port 0 tx
                                core 4 handles port 1 rx & core 5,6 handles port 1 tx
[1-2:3].0, [4-5:6].1, ...     - core 1,2 handles port 0 rx, core 3 handles port 0 tx
                                core 4,5 handles port 1 rx & core 6 handles port 1 tx
[1-2:3-5].0, [4-5:6/8].1, ... - core 1,2 handles port 0 rx, core 3,4,5 handles port 0 tx
                                core 4,5 handles port 1 rx & core 6,8 handles port 1 tx
[1:2].[0:0-7], [3:4].[1:0-7], - core 1 handles port 0 rx, core 2 handles ports 0-7 tx
                                 core 3 handles port 1 rx & core 4 handles port 0-7 tx
BTW: you can use "{}" instead of "[]" as it does not matter to the syntax.
Grouping can use {} instead of [] if required.
```
