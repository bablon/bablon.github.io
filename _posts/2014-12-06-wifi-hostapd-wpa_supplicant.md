---
layout: post
title:  "Wi-Fi开发－hostapd And wap_supplicant"
date:   2014-12-06 20:45:00
categories: Wi-Fi Openwrt
---

最近做Wi-Fi开发，发现极路由和小K插座使用Openwrt这个有名的用于路由器的开源Linux系统。Openwrt Wi-Fi功能由hostapd和wpa_supplicant负责，hostapd提供wifi AP功能，wpa_supplicant提供Station功能。由于还没有合适的开发板，先在笔记本上使用这两个程序来实现手动连接wifi，创建AP，初步了解其使用。

我的无线网卡是Intel Wireless-N 2200 (rev c4)，其支持的特性使用iw list查看：

	linux > iw list
	...
	Supported interface modes:
		 * IBSS
		 * managed
		 * AP
		 * AP/VLAN
		 * monitor	
	...
	HT Capability overrides:
		 * MCS: ff ff ff ff ff ff ff ff ff ff
		 * maximum A-MSDU length
		 * supported channel width
		 * short GI for 40 MHz
		 * max A-MPDU length exponent
		 * min MPDU start spacing	
	...

因为网卡支持的特性不支持wds，所以无法使用hostapd的wds功能。

wpa_supplication
================

连接到我的路由器，网络名是wifitest, 密钥为psktest。

	linux > yum install hostapd
	linux > cat > wpa_supplicant.conf << EOF
	network={
		ssid="wifitest"
		scan_ssid=1
		key_mgmt=WPA-PSK
		psk="psktext"
	}
	EOF
	linux > wpa_supplicant -Dwext -iwlp2s0 -c wpa_supplicant.conf

连接成功后，使用dhclient获取IP地址

	linux > dhclient wlp2s0


	
hostapd
=======

创建AP babylon, 密钥babylon

	linux > yum install hostapd
	linux > cat > hostapd.conf << EOF
	ctrl_interface=/var/run/hostapd
	ctrl_interface_group=wheel

	macaddr_acl=0
	auth_algs=1
	ignore_broadcast_ssid=0

	wpa=2
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP

	driver=nl80211

	interface=wlp2s0
	hw_mode=g
	ieee80211n=1
	channel=1
	ssid=babylon
	wpa_passphrase=babylon
	EOF
	linux > hostapd hostapd.conf

dnsmasq是一个轻量极的DNS和DHCP服务器，使用它来提供DHCP服务

	linux > yum install dnsmasq
	linux > cat > dnsmasq.conf << EOF
	interface=wlp2s0
	listen-address=192.168.42.1
	dhcp-range=192.168.42.100,192.168.42.200,12h
	dhcp-option=3,192.168.42.1
	dhcp-option=6,192.168.42.1
	linux > ifconfig wlp2s0 192.168.42.1 network mask 255.255.255.0
	linux > ifconfig up
	linux > dnsmasq -C dnsmasq.conf

可使用手机连接到该网络，并获取IP地址
