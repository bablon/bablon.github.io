---
layout: post
title:  "ns-3的概念抽象及实现"
date:   2014-09-10 00:29:32
categories: ns-3
---

译自[ns-3 official tutorial](http://www.nsnam.org/docs/release/3.20/tutorial/ns-3-tutorial.pdf)，
详见相关章节

Node
----
ns-3网络模拟器的基本设备抽象称为`node`，用C++的类`Node`表示。Node类提供方法来管理模拟设备  
的表示。Node类似于计算机，可以添加应用程序，协议栈和网卡。

Application
-----------
ns-3用户程序的基本抽象是`application`，用C++的类`Application`表示。开发人员基于此类创建　
新的applications。

Channel
-------
现实世界中数据流传输的媒介称为channel。ns-3基本通信子网抽象称为`channel`， 用`Channel`  
类表示，该类提供方法管理子网通信和连接结点。开必人员可基于此类开发派生Channel类来模拟有线，  
以太网或三维无线网络。

Net Device
----------
ns-3的`net device`抽象包括软件驱动和模拟硬件，Node安装net device才能与其它Nodes通过Channels
通信。通过添加多个net device，Node可连接到多个Channes。net device抽象用类`NetDevice`表示，
用来管理到Node和Channel的连接。

NetDevice派生实例：

* Node <--> CsmaNetDevice <--> CsmaChannel
* Node <--> PointToPointNetDevice <--> PointToPointChannel
* Node <--> WifiNetDevice <--> WifiChannel

Topology Helpers
----------------
ns-3中Nodes添加NetDevices，NetDevices绑定Channels，分配IP地址等任务是常用的操作，`topology helpers`
提供一种拓扑容器的功能来整合这些操作。

A First ns-3 Scrip
------------------
{% highlight c %}
/* -*- Mode:C++; c-file-style:"gnu"; indent-tabs-mode:nil; -*- */
/*
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation;
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, write to the Free Software
 * Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 */

#include "ns3/core-module.h"
#include "ns3/network-module.h"
#include "ns3/internet-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/applications-module.h"

using namespace ns3;

NS_LOG_COMPONENT_DEFINE ("FirstScriptExample");

int
main (int argc, char *argv[])
{
  CommandLine cmd;
  cmd.Parse (argc, argv);

  Time::SetResolution (Time::NS);
  LogComponentEnable ("UdpEchoClientApplication", LOG_LEVEL_INFO);
  LogComponentEnable ("UdpEchoServerApplication", LOG_LEVEL_INFO);

  NodeContainer nodes;
  nodes.Create (2);

  PointToPointHelper pointToPoint;
  pointToPoint.SetDeviceAttribute ("DataRate", StringValue ("5Mbps"));
  pointToPoint.SetChannelAttribute ("Delay", StringValue ("2ms"));

  NetDeviceContainer devices;
  devices = pointToPoint.Install (nodes);

  InternetStackHelper stack;
  stack.Install (nodes);

  Ipv4AddressHelper address;
  address.SetBase ("10.1.1.0", "255.255.255.0");

  Ipv4InterfaceContainer interfaces = address.Assign (devices);

  UdpEchoServerHelper echoServer (9);

  ApplicationContainer serverApps = echoServer.Install (nodes.Get (1));
  serverApps.Start (Seconds (1.0));
  serverApps.Stop (Seconds (10.0));

  UdpEchoClientHelper echoClient (interfaces.GetAddress (1), 9);
  echoClient.SetAttribute ("MaxPackets", UintegerValue (1));
  echoClient.SetAttribute ("Interval", TimeValue (Seconds (1.0)));
  echoClient.SetAttribute ("PacketSize", UintegerValue (1024));

  ApplicationContainer clientApps = echoClient.Install (nodes.Get (0));
  clientApps.Start (Seconds (2.0));
  clientApps.Stop (Seconds (10.0));

  Simulator::Run ();
  Simulator::Destroy ();
  return 0;
}
{% endhighlight %}

运行及输出：
{% highlight text %}
[galileo@localhost ns-3.20]$ ./waf --run first
Waf: Entering directory `/home/galileo/tool/ns-allinone-3.20/ns-3.20/build'
Waf: Leaving directory `/home/galileo/tool/ns-allinone-3.20/ns-3.20/build'
'build' finished successfully (13.193s)
At time 2s client sent 1024 bytes to 10.1.1.2 port 9
At time 2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time 2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time 2.00737s client received 1024 bytes from 10.1.1.2 port 9
{% endhighlight %}
