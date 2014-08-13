---
layout: post
title:  "Linux使用C实现DNS查询"
date:   2014-08-13 16:32:56
categories: c
---

## DNS查询
通过主机名查找主机IP地址称为DNS查询，gethostbyname和getaddrinfo函数均具有DNS查询功能。
本文则根据查询机制实现这一功能。向DNS服务器发送DNS查询请求并接收回复，然后从回复中解析出IP地址。

## DNS数据报
RFC 1035定义DNS消息结构如下：

    +-------------------+
    | Header            |
    +-------------------+
    | Question          | the question for the name server
    +-------------------+
    | Answer            | RRs answering the question
    +-------------------+
    | Authority         | RRs pointing toward an authority
    +-------------------+
    | Additional        | RRs holding additional information
    +-------------------+

查询消息不使用answer，authority和additional。DNS查询有不同的目的，除了获取主机IP地址，还可以获取指定域的邮件交换/服务器等等。
首先发送带有主机名的请求，使用UDP协议，DNS服务器端口通常是53。下一步是接收带对应的回复。所有类型的查询和响应数据按上图组织。
DNS回复数据头部含有资源记录（Resource Record）answer、authority和addition数量信息。

## DNS头部

    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     ID                        |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |QR|  Opcode   |AA|TC|RD|RA| Z      |  RCODE    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                  QDCOUNT                      |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                  ANCOUNT                      |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                  NSCOUNT                      |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                  ARCOUNT                      |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

ID - 16位消息标识符，由client生成，回复ID使用请求ID，以使client正确匹配对应的回复。
QR - 1bit，指示消息是查询0还是应答1。
OPCODE - 4bit，指定查询类型，0表示标准查询。
AA - 1bit，权威应答标志，仅作用于应答。表示收到应答是否具有权威性。
TC - 

http://www.binarytides.com/dns-query-code-in-c-with-linux-sockets/

## c代码
