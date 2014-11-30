---
layout: post
title:  "s3c6410开发－nfs挂载文件系统"
date:   2014-11-30 21:00:00
categories: ns-3
---

在公司维护过一段时间嵌入式底层平台系统，修复bug，优化性能，然而从未使用busybox完整地制作文件系统总觉得有些遗憾。不久前朋友搬家留下一个新的s3c6410开发板，正好可以使用它搭建一个开发系统，使用NFS挂载busybox制作的文件系统。

u-boot s3c-1.1.6
================

u-boot使用s3c-1.1.6，已编译好，由于原开发板上没有u-boot，所以不能使用u-boot来烧写我编译的u-boot，只有使用OpenJTAG工具。安装openocd，通过openocd来烧写u-boot。

    yum install openocd # the installed version is 0.7
    openocd -f interface/myjtag.cfg -f board/mini6410.cfg

myjtag.cfg是基于jtagkey.cfg修改的内容如下

    interface ft2232
    ft2232_device_desc "USB<=>JTAG&RS232 A"
    ft2232_layout jtagkey
    ft2232_vid_pid 0x1457 0x5118

    jtag_rclk 500

在另一个窗口登录到openocd进行操作

    telnet localhost 4444
    halt
    nand probe
    nand write 0 u-boot.bin 0 # wait for a long time
    init

kernel 3.14.25
==============

内核选择长期支持版本中的最高版本3.14.25，支持我的开发板的相关代码已合入内核

    make ARCH=arm s3c6400_defconfig

生成默认配置，要支持NFS，还需要添加dm9000网卡驱动和NFS配置，其他功能根据需要进行裁剪。

    make ARCH=arm menuconfig
    
    # nfs configuration
    --- Networking support
          Networking options  --->
            [*] TCP/IP networking
            [ ]   IP: multicasting
            [ ]   IP: advanced router
            [*]   IP: kernel level autoconfiguration
            [*]     IP: DHCP support
            [ ]     IP: BOOTP support
            [ ]     IP: RARP support
    
    --- Network File Systems
    <*>   NFS client support
    <*>     NFS client support for NFS version 2
    <*>     NFS client support for NFS version 3
    [ ]       NFS client support for the NFSv3 ACL protocol extension
    < >     NFS client support for NFS version 4
    [ ]     Provide swap over NFS support
    [*]   Root file system on NFS

    # dm9000 driver configuration
    Device Drivers  --->
      [*] Network device support  --->
      [*]   Ethernet driver support  --->
        <*>   DM9000 support
        [ ]     Force simple NSR based PHY polling  

busybox 1.22.1
==============

使用静态编译并安装到nfs共享目录，文件系统中其它必要目录的创建及配置文件的创建参考
TI技术文档[Configure the New Target Root File System][ti-rootfs-help]。

[ti-rootfs-help]:http://processors.wiki.ti.com/index.php/Creating_a_Root_File_System_for_Linux_on_OMAP35x#Configure_the_New_Target_Root_File_System

start
=====

开发板启动到u-boot，设置内核启动参数为

    setenv bootargs console=ttySAC0,115200 mem=128M root=/dev/nfs rw nfsroot=192.168.41.100:/home/galileo/nfs,nolock ip=192.168.41.90:192.168.41.100:192.168.41.1:255.255.255.0
    saveenv
    tftp 50008000 zImage
    nand write 50008000 80000 260000
    bootm 50008000

nfsroot参数，要加上选项nolock，不加会挂载不成功。

启动记录

	U-Boot 1.1.6 (Mar 17 2013 - 01:14:30) for LDD6410
	****************************************
	**    LDD-S3C6410 Nand boot v0.1      **
	****************************************

	CPU:     S3C6410@532MHz
		 Fclk = 532MHz, Hclk = 133MHz, Pclk = 66MHz, Serial = CLKUART (SYNC Mode) 
	Board:   LDD6410
	DRAM:    128 MB
	Flash:   0 kB
	NAND:    256 MB 
	In:      serial
	Out:     serial
	Err:     serial
	Hit any key to stop autoboot:  0 

	NAND read: device 0 offset 0x80000, size 0x260000
	 2490368 bytes read: OK
	Boot with zImage

	Starting kernel ...

	Uncompressing Linux... done, booting the kernel.
	Booting Linux on physical CPU 0x0
	Linux version 3.14.25 (galileo@localhost.localdomain) (gcc version 4.7.2 (Sourcery CodeBench Lite 2012.09-64) )4
	CPU: ARMv6-compatible processor [410fb766] revision 6 (ARMv7), cr=00c5387d
	CPU: PIPT / VIPT nonaliasing data cache, VIPT nonaliasing instruction cache
	Machine: MINI6410
	Memory policy: Data cache writeback
	CPU S3C6410 (id 0x36410101)
	CPU: found DTCM0 8k @ 00000000, not enabled
	CPU: moved DTCM0 8k to fffe8000, enabled
	CPU: found DTCM1 8k @ 00000000, not enabled
	CPU: moved DTCM1 8k to fffea000, enabled
	CPU: found ITCM0 8k @ 00000000, not enabled
	CPU: moved ITCM0 8k to fffe0000, enabled
	CPU: found ITCM1 8k @ 00000000, not enabled
	CPU: moved ITCM1 8k to fffe2000, enabled
	Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 32512
	Kernel command line: console=ttySAC0,115200 mem=128M root=/dev/nfs rw nfsroot=192.168.41.100:/home/galileo/nfs,0
	PID hash table entries: 512 (order: -1, 2048 bytes)
	Dentry cache hash table entries: 16384 (order: 4, 65536 bytes)
	Inode-cache hash table entries: 8192 (order: 3, 32768 bytes)
	Memory: 125296K/131072K available (3005K kernel code, 253K rwdata, 904K rodata, 144K init, 237K bss, 5776K rese)
	Virtual kernel memory layout:
	    vector  : 0xffff0000 - 0xffff1000   (   4 kB)
	    DTCM    : 0xfffe8000 - 0xfffec000   (  16 kB)
	    ITCM    : 0xfffe0000 - 0xfffe4000   (  16 kB)
	    fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
	    vmalloc : 0xc8800000 - 0xff000000   ( 872 MB)
	    lowmem  : 0xc0000000 - 0xc8000000   ( 128 MB)
	    modules : 0xbf000000 - 0xc0000000   (  16 MB)
	      .text : 0xc0008000 - 0xc03d9628   (3910 kB)
	      .init : 0xc03da000 - 0xc03fe32c   ( 145 kB)
	      .data : 0xc0400000 - 0xc043f560   ( 254 kB)
	       .bss : 0xc0440000 - 0xc047b724   ( 238 kB)
	SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
	NR_IRQS:390
	S3C6410 clocks: apll = 532000000, mpll = 532000000
		epll = 24000000, arm_clk = 532000000
	VIC @f6000000: id 0x00041192, vendor 0x41
	VIC @f6010000: id 0x00041192, vendor 0x41
	sched_clock: 32 bits at 33MHz, resolution 30ns, wraps every 129171948513ns
	Console: colour dummy device 80x30
	Calibrating delay loop... 528.79 BogoMIPS (lpj=2643968)
	pid_max: default: 32768 minimum: 301
	Mount-cache hash table entries: 1024 (order: 0, 4096 bytes)
	Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes)
	CPU: Testing write buffer coherency: ok
	Setting up static identity map for 0x502d9a80 - 0x502d9adc
	VFP support v0.3: implementor 41 architecture 1 part 20 variant b rev 5
	NET: Registered protocol family 16
	DMA: preallocated 256 KiB pool for atomic coherent allocations
	MINI6410: Option string mini6410=0
	MINI6410: selected LCD display is 480x272
	S3C6410: Initialising architecture
	bio: create slab <bio-0> at 0
	pl08xdmac dma-pl080s.0: initialized 8 virtual memcpy channels
	pl08xdmac dma-pl080s.0: initialized 16 virtual slave channels
	pl08xdmac dma-pl080s.0: DMA: PL080s rev1 at 0x75000000 irq 73
	pl08xdmac dma-pl080s.1: initialized 8 virtual memcpy channels
	pl08xdmac dma-pl080s.1: initialized 12 virtual slave channels
	pl08xdmac dma-pl080s.1: DMA: PL080s rev1 at 0x75100000 irq 74
	usbcore: registered new interface driver usbfs
	usbcore: registered new interface driver hub
	usbcore: registered new device driver usb
	Switched to clocksource samsung_clocksource_timer
	NET: Registered protocol family 2
	TCP established hash table entries: 1024 (order: 0, 4096 bytes)
	TCP bind hash table entries: 1024 (order: 2, 20480 bytes)
	TCP: Hash tables configured (established 1024 bind 1024)
	TCP: reno registered
	UDP hash table entries: 256 (order: 1, 12288 bytes)
	UDP-Lite hash table entries: 256 (order: 1, 12288 bytes)
	RPC: Registered named UNIX socket transport module.
	RPC: Registered udp transport module.
	RPC: Registered tcp transport module.
	RPC: Registered tcp NFSv4.1 backchannel transport module.
	futex hash table entries: 256 (order: 0, 7168 bytes)
	Installing knfsd (copyright (C) 1996 okir@monad.swb.de).
	ROMFS MTD (C) 2007 Red Hat, Inc.
	io scheduler noop registered
	io scheduler deadline registered
	io scheduler cfq registered (default)
	s3c-fb s3c-fb: window 0: fb 
	Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
	s3c6400-uart.0: ttySAC0 at MMIO 0x7f005000 (irq = 69, base_baud = 0) is a S3C6400/10
	console [ttySAC0] enabled
	s3c6400-uart.1: ttySAC1 at MMIO 0x7f005400 (irq = 70, base_baud = 0) is a S3C6400/10
	s3c6400-uart.2: ttySAC2 at MMIO 0x7f005800 (irq = 71, base_baud = 0) is a S3C6400/10
	s3c6400-uart.3: ttySAC3 at MMIO 0x7f005c00 (irq = 72, base_baud = 0) is a S3C6400/10
	brd: module loaded
	loop: module loaded
	s3c24xx-nand s3c6400-nand: failed to get clock
	s3c24xx-nand: probe of s3c6400-nand failed with error -2
	dm9000 dm9000: read wrong id 0x01010101
	dm9000 dm9000: eth%d: Invalid ethernet MAC address. Please set using ifconfig
	eth0: dm9000b at c885a000,c885c004 IRQ 108 MAC: 02:a2:c3:25:dc:e0 (random)
	ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
	ohci-s3c2410: OHCI S3C2410 driver
	s3c2410-ohci s3c2410-ohci: OHCI Host Controller
	s3c2410-ohci s3c2410-ohci: new USB bus registered, assigned bus number 1
	s3c2410-ohci s3c2410-ohci: irq 79, io mem 0x74300000
	s3c2410-ohci s3c2410-ohci: init err (00000000 0000)
	s3c2410-ohci s3c2410-ohci: can't start
	s3c2410-ohci s3c2410-ohci: startup error -75
	s3c2410-ohci s3c2410-ohci: USB bus 1 deregistered
	s3c2410-ohci: probe of s3c2410-ohci failed with error -75
	mousedev: PS/2 mouse device common for all mice
	i2c /dev entries driver
	sdhci: Secure Digital Host Controller Interface driver
	sdhci: Copyright(c) Pierre Ossman
	s3c-sdhci s3c-sdhci.0: clock source 0: mmc_busclk.0 (133000000 Hz)
	s3c-sdhci s3c-sdhci.0: clock source 2: mmc_busclk.2 (24000000 Hz)
	mmc0: SDHCI controller on samsung-hsmmc [s3c-sdhci.0] using ADMA
	s3c-sdhci s3c-sdhci.1: clock source 0: mmc_busclk.0 (133000000 Hz)
	s3c-sdhci s3c-sdhci.1: clock source 2: mmc_busclk.2 (24000000 Hz)
	mmc1: SDHCI controller on samsung-hsmmc [s3c-sdhci.1] using ADMA
	usbcore: registered new interface driver usbhid
	usbhid: USB HID core driver
	TCP: cubic registered
	drivers/rtc/hctosys.c: unable to open rtc device (rtc0)
	dm9000 dm9000 eth0: link down
	dm9000 dm9000 eth0: link down
	IP-Config: Complete:
	     device=eth0, hwaddr=02:a2:c3:25:dc:e0, ipaddr=192.168.41.90, mask=255.255.255.0, gw=192.168.41.1
	     host=192.168.41.90, domain=, nis-domain=(none)
	     bootserver=192.168.41.100, rootserver=192.168.41.100, rootpath=
	dm9000 dm9000 eth0: link up, 100Mbps, full-duplex, lpa 0x4DE1
	VFS: Mounted root (nfs filesystem) on device 0:10.
	Freeing unused kernel memory: 144K (c03da000 - c03fe000)

	    System initialization...

	    Hostname       : S3C6410EVM
	    Filesystem     : v1.0.0


	    Kernel release : Linux 3.14.25
	    Kernel version : #4 Thu Nov 27 23:12:30 CST 2014

	 Mounting /proc             : [SUCCESS]
	 Mounting /sys              : [SUCCESS]
	 Mounting /dev              : [SUCCESS]
	 Mounting /dev/pts          : [SUCCESS]
	 Enabling hot-plug          : [SUCCESS]
	 Populating /dev            : [SUCCESS]
	 Mounting other filesystems : [SUCCESS]
	 Starting syslogd           : [SUCCESS]
	 Starting telnetd           : [SUCCESS]

	System initialization complete.

	Please press Enter to activate this console. 
	/ # ps
	PID   USER     TIME   COMMAND
	    1 root       0:00 init
	    2 root       0:00 [kthreadd]
	    3 root       0:00 [ksoftirqd/0]
	    4 root       0:00 [kworker/0:0]
	    5 root       0:00 [kworker/0:0H]
	    6 root       0:00 [kworker/u2:0]
	    7 root       0:00 [khelper]
	    8 root       0:00 [netns]
	    9 root       0:00 [kworker/u2:1]
	  141 root       0:00 [writeback]
	  144 root       0:00 [bioset]
	  146 root       0:00 [kblockd]
	  197 root       0:00 [khubd]
	  292 root       0:00 [rpciod]
	  293 root       0:00 [kworker/0:1]
	  301 root       0:00 [kswapd0]
	  350 root       0:00 [fsnotify_mark]
	  357 root       0:00 [nfsiod]
	  957 root       0:00 [kpsmoused]
	  964 root       0:00 [kworker/u2:2]
	  966 root       0:00 [irq/111-s3c-sdh]
	  985 root       0:00 [deferwq]
	  986 root       0:00 [kworker/0:2]
	  987 root       0:00 [kworker/0:3]
	 1010 root       0:00 /usr/sbin/telnetd
	 1013 root       0:00 -/bin/ash
	 1018 root       0:00 ps
	/ # 

