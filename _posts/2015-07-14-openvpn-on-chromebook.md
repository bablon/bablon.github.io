---
layout: post
title:  "OpenVPN On Chromebook"
date:   2015-07-14 13:30:00
categories: openvpn
---

前些天用openvpnas搭建了一个openvpn服务器，openvpnas全称OpenVPN Access Server，是openvpn的商业化解决方案，不需要复杂的配置，安装完成后就能使用。通过web来进行管理，修改配置和添加用户，用户登录后下载用户的配置文件给对应平台的客户端。没有许可证时同时连接的客户端不超过2个，个人在此范围内可自由使用。所以它主要面向的是企业用户。

Chromebook原生支持L2TP和OpenVPN, openvpn的配置比较灵活，chromeos支持的并不友好，它只支持用户名和密码的方式。直接使用ovpn配置文件的方式不支持，我并不想打开开发者模式，希望在用户层面就能够得到解决。chromeos有一个内部地址可以导入ONC配置文件，ONC配置文件是chromeos内部使用的网络配置文件,json格式，全称Open Network Configuration。通过它可以定义多种网络配置，用户可方便地在不同配置间切换。目前还没有工具生成或者转换成onc配置，只能手工将ovpn配置转换成onc配置文件，这需要对两个配置都要有足够的理解。

# 参考资源

* [OpenVPN on ChromeOS Documentation][openvpn_on_chromeos]
* [Open Network Configuration Format][onc_format]
* [Online UUID Generator][uuid_generator]

[openvpn_on_chromeos]: https://docs.google.com/document/d/18TU22gueH5OKYHZVJ5nXuqHnk2GN6nDvfu2Hbrb4YLE/pub#h.buf7fpkgt9c8
[onc_format]: http://src.chromium.org/chrome/trunk/src/components/onc/docs/onc_spec.html#sec_OpenVPN_connections_and_types
[uuid_generator]: https://www.uuidgenerator.net/

# 准备证书文件

ovpn文件支持内嵌证书，`ca`标签之间的内容认证机构证书，提取并保存为ca.crt。`cert`和`key`分别是用户证书和私钥，保存为client.crt和client.key。chromeos仅支持PKCS12格式的用户证书，要将client.crt和client.key格式转换成pkcs12。
使用openssl命令来生成pkcs12格式的用户证书，client.p12。

    openssl pkcs12 -export -in client.crt -inkey client.key -certfile ca.crt -name MyClient -out client.p12
    
`tls-auth`标签之间是tls授权使用的静态key，保存为ta.key。

# 导入证书文件

打开地址`chrome://settings/certificates`，选择授权中心，导入ca.crt证书。选择您的证书，使用`导入并绑定到设备`导入client.p12文件。

# 准备ONC文件

根据参考资源1所提供的文件实例，结合自己的ovpn文件进行修改。

# 导入ONC文件

打开地址chrome://net-internals， 选择ChromeOS，在`Import ONC file`标题下，执行导入操作。

# ONC配置文件

Here is my onc and ovpn file on [github][gist-onc].

[gist-onc]: https://gist.github.com/bablon/a15a435315348b4eccfa

`RemoteCertTLS`的默认值是`server`，我在开始配置的过程中，总是配置失败，查看日志一直是

    ssl3_get_server_certificate:certificate verify failed

错误。后来仔细比较内部配置输出和ovpn配置的差异，将`RemoteCertTLS`设为`none`，取得配置成功。

证书的格式是`X509`，内容占一行，要将ca.crt文件中的多行合并成一行。

    grep -v '-' ca.crt | perl -p -e 's/\n/\\n/' -
    
tls授权的静态key和json key是`TLSAuthContents`, 它的内容也是占一行，但不同于证书的格式。

    grep -v '-' ca.crt | perl -p -e 's/\n/\\n/' -
    
GUID字段是一个全局唯一的字符串，可以用UUID生成器来生成。
    
# 调试方法

内部地址: chrome://system，用来查看系统信息。`network-services`记录服务启动信息。`netlog`记录了网络工作的日志。如果出现错误可以在这两个地址进行错误信息的查找。
