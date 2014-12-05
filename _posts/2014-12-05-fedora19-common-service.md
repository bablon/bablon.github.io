---
layout: post
title:  "Fedora 19常用服务安装与配置"
date:   2014-12-05 20:00:00
categories: fedora
---

是时候总结一下这些常用服务的安装与配置了，今天配置telnet服务又折腾了一番，最后简单地解决了，在使用systemd启动服务后，telnet server不用添加修改配置文件。

xinetd
======
telnet server和tftp server由xinetd守护进程启动,需要先安装

	yum install xinetd
	systemctl start xinetd.service

	# boot start
	systemctl enable xinetd.service

Telnet Server
=============

	yum install telnet-server

	# default configuration
	systemctl start telnet.socket

	# /etc/xinetd.d/telnet reference
	service telnet
	{
		flags = REUSE
		socket_type = stream
		wait = no
		user = root
		server = /usr/sbin/in.telnetd
		disable = no
		log_on_failure += USERID
	}

TFTP Server
===========

	yum install tftp-server

	# tftp config file /etc/xinet.d/tftp
	service tftp
	{
		socket_type		= dgram
		protocol		= udp
		wait			= yes
		user			= root
		server			= /usr/sbin/in.tftpd
		server_args		= -s /tftpboot/path -c
		disable			= yes
		per_source		= 11
		cps			= 100 2
		flags			= IPv4
	}
	# the disable is no for default

	# start tftp
	systemctl start tftp.service or
	systemctl start tftp.socket
	# 如果disable为no, tftp.service启动不成功,
	# tftp.socket能够启动成功，tftp目录为/var/lib/tftpboot

NFS Server
==========
推荐一个网站[Server World][server-world-url], 里面不少服务的安装与配置，NFS Server和FTP Server的配置参考该网站。

[server-world-url]:http://www.server-world.info/en/note?os=Fedora_19&p=nfs&f=1

	yum install nfs-utils

	# /etc/exports
	/home 10.0.0.0/24(rw,sync,no_root_squash,no_all_squash)
	# /home shared directory
	# 10.0.0.0/24 range of networks NFS permits accesses
	# rw writable
	# sync synchronize
	# no_root_squash enable root privilege
	# no_all_squash enable users' authority

	systemctl start rpcbind.service
	systemctl start nfs-server.service
	systemctl start nfs-lock.service
	systemctl start nfs-idmap.service

FTP Server
==========

	yum -y install vsftpd

	vi /etc/vsftpd/vsftpd.conf
	# line 12: no anonymous
	anonymous_enable= NO

	# line 82,83: uncomment ( allow ascii mode )
	ascii_upload_enable=YES
	ascii_download_enable=YES

	# line 100, 101: uncomment ( enable chroot )
	chroot_local_user=YES
	chroot_list_enable=YES

	# line 103: uncomment ( specify chroot list )
	chroot_list_file=/etc/vsftpd/chroot_list

	# line 109: uncomment
	ls_recurse_enable=YES

	# line 114: change ( if use IPv4 )
	listen= YES

	# line 123: change ( if use IPv4, IPv6 is off )
	listen_ipv6= NO

	# add at the last line
	# specify root directory ( if don't specify, users' home directory become FTP home directory)
	local_root=public_html

	# use localtime
	use_localtime=YES

	# turn off for seccomp filter ( if you cannot login, add this line )
	seccomp_sandbox=NO

	vim /etc/vsftpd/chroot_list
	# add users you allow to move over their home directory
	fedora

	systemctl start vsftpd.service
