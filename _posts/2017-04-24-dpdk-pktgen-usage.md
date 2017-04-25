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

```shell
Where the options are:
    -h:  显示上面显示的使用/帮助信息.

    -s   P:file: PCAP报文文件路径. P端口号，暂时不知道什么用.
    
    -f   filename: pkt脚本或者lua脚本文件路径，定制化执行. See Running Script Files.

    -l   filename: 日志文件路径.
    
    -P:  是否开始混杂模式.
    
    -G:  开启socket支持使用默认的服务器地址localhost:0x5606. See Socket Support for Pktgen.
    
    -g   address: 和-G差不多但是可以配置IP和PORT. See Socket Support for Pktgen.
    
    -T:  开始彩色输出 in VT100
    
    -N:  开启NUMA
    
    -m   <string>: DPDK网卡和逻辑核之间的矩阵映射. 映射格式使用BNF-like语法，如下：
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

## 3. 运行时命令

### 3.1 help 帮助命令

```shell
Pktgen> help

set <portlist> <xxx> value    - Set a few port values
save <path-to-file>           - Save a configuration file using the
                                filename
load <path-to-file>           - Load a command/script file from the
                                given path
...
```
运行时命令如下所述。

几个命令采用常见的参数，如：

* portlist: A list of ports such as <font color=DeepPink>2,4,6-9,12</font> or the word <font color=DeepPink>all</font>.
* state: This is usually <font color=DeepPink>on</font> or <font color=DeepPink>off</font> but will also accept <font color=DeepPink>enable</font> or <font color=DeepPink>disable</font>.

### 3.2 set 网卡设置

set命令主要用于设置端口的信息： 

    set <portlist> <command> value
    
    <portlist>:  端口的列表.
    <command>:   下面的某个字段:
        count:   发送报文数量.
        size:    发送报文大小.
        rate:    发送报文速率百分比.
        burst:   批量收发报文数量.
        sport:   TCP源端口号.
        dport:   TCP目的端口号.
        prime:   Set the number of packets to send on prime command.
        seqCnt:  Set the number of packet in the sequence to send.
        dump:    Dump the next <value> received packets to the screen.

```shell
For example:

    Pktgen> set all seqCnt 1
```
set命令也能够设置报文的MAC地址

    set mac <portlist> etheraddr

set命令也能够设置报文的IP地址

    set ip src|dst <portlist> ipaddr


### 3.3 seq 报文设置

seq命令主要设置往卡上面发送报文的信息：

        seq <seq#> <portlist> dst-Mac src-Mac dst-IP src-IP
                              sport dport ipv4|ipv6|vlan udp|tcp|icmp vid pktsize
        
        <seq#>:     报文序列号.
        <portlist>: 网卡编号.
        dst-Mac:    目的MAC地址.
        src-Mac:    源MAC地址.
        dst-IP:     目的IP地址.
        src-IP:     源IP地址. 确保sip有网络掩码，例如1.2.3.4/24.
        sport:      源端口.
        dport:      目的端口.
        IP:         IP层协议. One of ipv4|ipv6|vlan.
        Transport:  传输层协议. One of udp|tcp|icmp.
        vid:        VlanID.
        pktsize:    报文大小.

### 3.4 save 保存配置文件

save命令主要保存当前的配置到配置文件中：

    save <path-to-file>

### 3.5 load 加载配置文件

load命令主要加载一个配置文件从文件中：

    load <path-to-file>

大多数情况用于加载由save命令保存的配置文件.

### 3.6 ppp 显示的的ports数量

ppp(ports per page)命令设置每一页显示的的ports数量：

    ppp [1-6]

### 3.7 icmp.echo ICMP回应

icmp.echo命令开启或者关闭某个网卡的ICMP的回应功能：

    icmp.echo <portlist> <state>

[state值参考上面的说明](#31-help-帮助命令)

### 3.8 send 发送ARP

send命令发送一个ARP请求或者免费的ARP报文在设置的网卡上面：

    send arp req|grat <portlist>

### 3.9 mac_from_arp 获取MAC地址从ARP

mac_from_arp命令设置是否从ARP请求中获取MAC地址:

    mac_from_arp <state>

### 3.10 proto 传输层协议设置

proto命令在每个port上面设置报文的协议为UDP or TCP or ICMP:

    proto udp|tcp|icmp <portlist>

### 3.11 type IP层协议设置

The type command sets the packet type to IPv4 or IPv6 or VLAN:
type命令设置报文的三层协议为IPv4 or IPv6 or VLAN：

    type ipv4|ipv6|vlan <portlist>

### 3.12 geometry 显示设置

geometry命令设置设置显示的列和行(colxrow):

    geometry <geom>

### 3.13 capture 抓取报文

capture命令开启或者关闭网卡的报文抓取功能：

    capture <portlist> <state>

### 3.14 rxtap Rx TAP

rxtap命令开启或者关闭Rx tap接口:

    rxtap <portlist> <state>

### 3.15 txtap Tx TAP

txtap命令开启或者关闭Tx tap接口：

    txtap <portlist> <state>

### 3.16 vlan

vlan命令开启或者关闭发送带有VLAN ID的报文:

    vlan <portlist> <state>

### 3.17 vlanid

The vlanid command sets the VLAN ID for the portlist:

vlanid <portlist> <vlanid>

### 3.18 mpls

The mpls command enables/disables sending an MPLS entry in packets:

mpls <portlist> <state>

### 3.19 mpls_entry

The mpls_entry command sets the MPLS (Multiprotocol Label Switching) entry for the portlist (must be specified in hex):

mpls_entry <portlist> <entry>

### 3.20 qinq

The qinq command enables/disables sending a Q-in-Q header in packets:

qinq <portlist> <state>

### 3.21 qinqids

The qinqids command sets the Q-in-Q ID’s for the portlist:

qinqids <portlist> <id1> <id2>

### 3.22 gre

The gre command enables/disables GRE (Generic Routing Encapsulation) with IPv4 payload:

gre <portlist> <state>

### 3.23 gre_eth

The gre_eth command enables/disables GRE with Ethernet frame payload:

gre_eth <portlist> <state>

### 3.24 gre_key

The gre_key command sets the GRE key:

gre_key <portlist> <state>

### 3.25 pcap

The pcap command enables or disable sending pcap packets on a portlist:

pcap <portlist> <state>

### 3.26 pcap.show

The pcap.show command shows the PCAP information:

pcap.show
pcap.index

The pcap.index command moves the PCAP file index to the given packet number:

### 3.27 pcap.index
Where:

0 = rewind.
-1 = end of file.

### 3.28 pcap.filter

The pcap.filter command sets the PCAP filter string to filter packets on receive:

pcap.filter <portlist> <string>

### 3.29 script

The script command execute the Lua code in specified file:

script <filename>
See Running Script Files.

### 3.30 ping4

The ping4 command sends a IPv4 ICMP echo request on the given portlist:

ping4 <portlist>

### 3.31 page

The page command shows the port pages or configuration or sequence page:

page [0-7]|main|range|config|seq|pcap|next|cpu|rnd
Where:

[0-7]: Page of different ports.
main: Display page zero.
range: Display the range packet page.
config: Display the configuration page (reserved, not used).
pcap: Display the pcap page.
cpu: Display some information about the system CPU.
next: Display next page of PCAP packets.
sequence|seq: Display a set of packets for a given port. Note: use the port command, see below, to display a new port sequence.
rnd: Display the random bitfields of packets for a given port. Note: use the port command, see below, to display a new port sequence.
log: Display the log messages page.

### 3.32 port

The port command sets the sequence of packets to display for a given port:

port <number>

### 3.33 process

The process command enables or disables processing of ARP/ICMP/IPv4/IPv6 packets:

process <portlist> <state>

### 3.34 garp

The garp command enables or disables Gratuitous ARP packet processing and update MAC address:

garp <portlist> <state>

### 3.35 blink

The blink command blinks the link led on the given port list:

blink <portlist> <state>

### 3.36 rnd

The rnd command sets random mask for all transmitted packets from portlist:

rnd <portlist> <idx> <off> <mask>
Where:

idx: random mask slot.
off: offset in packets, where to apply mask.
mask: up to 32 bit long mask specification (empty to disable):
0: bit will be 0.
1: bit will be 1.
.: bit will be ignored (original value is retained).
X: bit will get random value.

### 3.37 theme

The theme command enables or disables the theme:

theme <state>
It also sets the color for item with foreground (fg) or background (bg) color and attribute value:

theme <item> <fg> <bg> <attr>
theme.show

The theme.show command lists the item strings, colors and attributes to the items:

theme.show
theme.save

The theme.save command saves the current color theme to a file:

theme.save <filename>

### 3.38 start

The start command starts transmitting packets:

start <portlist>

### 3.39 stop

The stop command stops transmitting packets:

stop <portlist>

### 3.40 str

The str command starts all ports transmitting:

str
A shortcut for start all.

### 3.41 stp

The stp command stops all ports from transmitting:

stp
A shortcut for stop all.

### 3.42 screen

The screen command stops/starts updating the screen and unlocks/locks the window:

screen stop|start

### 3.43 off

The off command is a screen off shortcut:

off
on

The on command screen on shortcut:

### 3.44 on

### 3.45 prime

The prime command transmits N packets on each port listed. See set prime command above:

prime <portlist>

### 3.46 delay

The delay command waits a number of milliseconds before reading or executing scripting commands:

delay milliseconds

### 3.47 sleep

The sleep command waits a number of seconds before reading or executing scripting commands:

sleep seconds

### 3.48 dev.list

The dev.list command shows the whitelist/blacklist/Virtual devices:

dev.list
pci.list

The pci.list command shows all the PCI devices:

pci.list

### 3.49 clear

The clear command clears the statistics:

clear <portlist>

### 3.50 clr

The clr command clears all statistics:

clr
A shortcut for clear all.

### 3.51 cls

The cls command clears the screen:

cls
A shortcut for clear all.

### 3.52 reset

The reset command resets the configuration to the default:

reset <portlist>

### 3.53 rst

The rst command resets the configuration for all ports:

rst
A shortcut for reset all.

### 3.54 help

The help command displays this help for runtime commands:

help
### 3.55 quit

The quit command quits the Pktgen program:

quit

### 3.56 dst.mac

The dst.mac command sets the destination MAC address start:

dst.mac start <portlist> etheraddr

### 3.57 src.mac

The src.mac command sets the source MAC address start:

src.mac start <portlist> etheraddr

### 3.58 src.ip

The src.ip command sets the source IP address properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

src.ip start <portlist> ipaddr
src.ip min <portlist> ipaddr
src.ip max <portlist> ipaddr
src.ip inc <portlist> ipaddr

### 3.59 dst.ip

The dst.ip command sets the destination IP address properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

dst.ip start <portlist> ipaddr
dst.ip min <portlist> ipaddr
dst.ip max <portlist> ipaddr
dst.ip inc <portlist> ipaddr

### 3.60 src.port

The src.port command sets the source port address properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

src.port start <portlist> value
src.port min <portlist> value
src.port max <portlist> value
src.port inc <portlist> value

### 3.61 dst.port

The dst.port command sets the source port address properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

dst.port start <portlist> value
dst.port min <portlist> value
dst.port max <portlist> value
dst.port inc <portlist> value

### 3.62 vlan.id

The vlan.id command sets the vlan id address properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

vlan.id start <portlist> value
vlan.id min <portlist> value
vlan.id max <portlist> value
vlan.id inc <portlist> value

### 3.63 pkt.size

The pkt.size command sets the packet size properties:

start: The start of the range.
min: The minimum value in range.
max The maximum value in range
inc: The increment.
For example:

pkt.size start <portlist> value
pkt.size min <portlist> value
pkt.size max <portlist> value
pkt.size inc <portlist> value

### 3.64 range

The range command enables or disables the given portlist for sending a range of packets:

range <portlist> <state>
