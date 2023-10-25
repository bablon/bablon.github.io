---
layout: post
title:  "TCP初始序列号"
date:   2023-09-15 20:30:00
categories: kernel
---

## 介绍

TCP报文中有Sequence Number和Acknowledgment Number两个序列号字段, Sequence Number表示TCP段中第一个字节的序列。Acknowledgment number用于应答，表示下一个期望收到的TCP段的Sequence number。由于每个连接的初始化序列号都不相同，本文结合协议和内核代码来探究TCP连接建立过程中的序列号初始化。下图是TCP建立连接的过程及序列号的一个示例。


    RFC9293:

        TCP Peer A                                           TCP Peer B
    1.  CLOSED                                               LISTEN
    2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED
    3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED
    4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED
    5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED

### 序列号计算方法

计算方程

    ISN = M + F(localip, localport, remoteip, remoteport, secretkey)

Ｍ是一个４毫秒计时器。Ｆ是一个有５个输入参数的伪随机函数, 根据TCP连接参数和密钥生成哈希值。

## 创建连接

服务端和客户端通过套接字完成三方握手来创建TCP连接。

### Client

客户端创建套接字，通过connect调用发送SYN请求和SYN-ACK的ACK应答，connect返回时连接完成。

	#define DESTIP		"x.y.z.w"
	#define DESTPORT	80

	int sock = socket(AF_INET, SOCK_STREAM, 0);
	struct sockaddr_in sin = { 
		.sin_family = AF_INET,
		.sin_addr.s_addr = inet_addr(DESTIP);
		.sin_port = htons(DESTPORT);
	};

	connect(sock, (struct sockaddr *)&sin, sizeof(sin));

### Server

服务端创建监听套接字，通过bind和listen监听指定端口上的请求。收到SYN请求后发送SYN-ACK包，收到ACK包后，创建连接套接字。accept获取连接套接字。

	#define LOCALPORT	80

	int sock = socket(AF_INET, SOCK_STREAM, 0);
	int conn_sock;
	struct sockaddr_in peer;
	socklen_t len = sizeof(peer);
	struct sockaddr_in sin = {
		.sin_family = AF_INET,
		.sin_addr.sin_addr = 0;
		.sin_port = htons(LOCALPORT);
	};

	bind(sock, (struct sockaddr *)&sin, sizeof(sin));
	listen(sock, 512);
	conn_sock = accept(sock, (struct sockaddr *)&peer, &len);

## 过程分析

内核负责套接字创建和状态切换，以及握手包的组装与发送。上面用户层使用的API会执行到内核层的处理函数，下面结合伪码对其中关键处理函数进行描述。


### SYN (SEQ = X, ACK = 0)

`tcp_v4_conn_connect`设置TCP状态为`TCP_SYN_SENT`， 计算初始序列号，然后创建和填充TCP报文并发送。序列号计算函数是`secure_tcp_seq`与ISN计算方程对应。

    int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
    {
        struct inet_sock *inet = inet_sk(sk);
        struct tcp_sock *tp = tcp_sk(sk);
        struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
        struct sk_buff *buff;
        struct tcphdr *th;

        tcp_set_state(sk, TCP_SYN_SENT);
        tp->write_seq = secure_tcp_seq(inet->saddr, inet->daddr,
                           inet->sport, usin->sin_port);
        tp->rcv_nxt = 0;

        buff = tcp_stream_alloc_skb(sk, sk->sk_allocation, true);
        skb_reserve(buff, MAX_TCP_HEADER);
        tcp_header_size = tcp_option_size + sizeof(struct tcphdr);
        skb_push(buff, tcp_header_size);

        TCP_SKB_CB(buff)->seq = tp->write_seq;

        th = (struct tcphdr *)skb->data;

        th->seq = tp->write_seq;
        th->ack_seq = tp->rcv_nxt;
        th->syn = 1;

        // send packet
        …

        tp->snd_nxt = tp->write_seq + 1;
    }

`net_securet_init`函数设置`net_secret`随机密钥，结合TCP连接参数进行哈希计算，再通过`seq_scale`函数与时间关联得到最终的序列号。

    static __always_inline void net_secret_init(void)
    {
        net_get_random_once(&net_secret, sizeof(net_secret));
    }

    static u32 seq_scale(u32 seq)
    {
        return seq + (ktime_get_real_ns() >> 6);
    }

    u32 secure_tcp_seq(__be32 saddr, __be32 daddr,
               __be16 sport, __be16 dport)
    {
        u32 hash;

        net_secret_init();
        hash = siphash_3u32((__force u32)saddr, (__force u32)daddr,
                    (__force u32)sport << 16 | (__force u32)dport,
                    &net_secret);
        return seq_scale(hash);
    }

### SYN+ACK (SEQ = Y, ACK = X + 1)

内核收到连接请求后，创建请求套接字，该请求套接字状态是`TCP_NEW_SYN_RECV`。同样调用`secure_tcp_seq`生成一个初始化序列号。内核组装一个合并应答和请求的SYN+ACK包，Acknowledgment Number是请求中的Sequence Number + 1。

    int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
    {
        u32 isn;
        struct sk_buff *skb;
        struct tcphdr *th;
        struct request_sock *req;

        req = reqsk_alloc(rsk_ops, sk, 1);
        req->__req_common.skc_state = TCP_NEW_SYN_RECV;

        isn = secure_tcp_seq(iphdr(skb)->daddr, iphdr(skb)->saddr,
                     tcphdr(skb)->dest, tcphdr(skb)->source);

        skb = alloc_skb(MAX_TCP_HEADER, GFP_ATOMIC);
        skb_reserve(skb, MAX_TCP_HEADER);

        skb_push(skb, tcp_header_size);
        th = (struct tcphdr *)skb→data;

        th->syn = 1;
        th->ack = 1;
        th->seq = isn;
        th->ack = htonl(ntohl(tcphdr(skb)->seq) + 1);

        ...
    }

### ACK (SEQ = X + 1, ACK = Y + 1)

客户端内核收到SYN+ACK包后，套接字状态切换到TCP\_ESTABLISHED。然后回复服务端的请求，Ack序列号是对端请求Seq+1, Seq序列号是初始化Seq+1。

    static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb,
                         const struct tcphdr *th)
    {
        struct sk_buff *skb;

        tcp_set_state(sk, TCP_ESTABLISHED);

        skb  = alloc_skb(MAX_TCP_HEADER, flag);
        skb_reserve(skb, MAX_TCP_HEADER);
        skb_push(skb, tcp_header_size);

        th = (struct tcphdr *)skb->data;

        th->ack = 1;
        th->seq = tp->snd_nxt;
        th->ack = htonl(ntohl(tcphdr(skb)->seq) + 1);
    }

### ACK Received

内核收到客户端应答，找到之前创建的请求套接字。基于该套接字，创建状态为`TCP_SYN_RECV`的连接套接字，处理后切换到`TCP_ESTABLISHED`，放入accept队列中供上层读取。

    int tcp_v4_rcv(struct sk_buff *skb)
    {
        struct sock sk = __inet_look_skb(.., skb, ...);

        // case TCP_NEW_SYN_RECV
        struct sock *newsk = sk_clone_lock(sk, priority);
        inet_sk_set_state(newsk, TCP_SYN_RECV);

        // tcp_rcv_state_process(nsk, skb);
        // case TCP_SYN_RECV;
        inet_sk_set_state(newsk, TCP_ESTABLISHED);
    }
