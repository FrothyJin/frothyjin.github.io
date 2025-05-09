---
title: VMware虚拟机网络配置
author: YooyoJin
date: 2025-03-07
category: Jekyll
layout: post
---

在使用虚拟机的过程中发现，我的虚拟机虽然可以正常上网，但是无法连接VPN。

于是通过上网查询后，通过以下方式完美解决了 

> 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。    
> [VMware Ubuntu虚拟机使用主机VPN配置](https://blog.csdn.net/nomoremorphine/article/details/138738065)


于是产生疑惑，虚拟机和主机是使用同一个网络吗？

## 1. 网络适配器配置信息解析

当我们在主机查询ip时可以获得关键大致如下信息：

``` cmd
无线局域网适配器 WLAN:
   连接特定的 DNS 后缀 . . . . . . . :
   IPv6 地址 . . . . . . . . . . . . : 2409:8a28:a6b:7854:5060:e594:73d3:c6ea
   临时 IPv6 地址. . . . . . . . . . : 2409:8a28:a6b:7854:3136:4ce0:c2ce:1e54
   本地链接 IPv6 地址. . . . . . . . : fe80::a823:c7db:4471:8df6%4
   IPv4 地址 . . . . . . . . . . . . : 192.168.5.3
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : fe80::5%4
                                       192.168.5.1

以太网适配器 VMware Network Adapter VMnet1:
   连接特定的 DNS 后缀 . . . . . . . :
   本地链接 IPv6 地址. . . . . . . . : fe80::ae29:fa20:d67a:1b2d%19
   IPv4 地址 . . . . . . . . . . . . : 192.168.115.1
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . :

以太网适配器 VMware Network Adapter VMnet8:
   连接特定的 DNS 后缀 . . . . . . . :
   本地链接 IPv6 地址. . . . . . . . : fe80::69bd:3c59:f8df:7f84%2
   IPv4 地址 . . . . . . . . . . . . : 192.168.80.1
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . :
```

目前我也不太清楚这种IP地址暴露会有什么风险，不过无所谓，不也是什么人物。

可以看到我的无线网络适配器已经成功获取了IPv4和IPv6地址，默认网关是`192.168.5.1`，这是我的路由器或接入点的IP地址。

然后可以看到两个以太网适配器VMnet1、VMnet8这是VMware虚拟网络适配器，通常用于主机与虚拟机之间的通信。VMnet1它有一个私有IPv4地址`192.168.115.1`，子网掩码为`255.255.255.0`；VMnet8IPv4地址`192.168.80.1`，子网掩码为`255.255.255.0`。没有配置默认网关，因为仅本地网络通信吧？

无论是我们是对其他仪器仪表的网络配置也好，通常也都绕不开这些。通常需要配置它们在同一网段内，IP地址的网络部分相同，子网掩码一致。使用ping命令测试连通性，保证通信正常。

## 2. VMware虚拟机三种网络模式

通过下面这篇文章也大概了解了下VMware虚拟机三种网络模式；

> 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> [VMware虚拟机三种网络模式：桥接模式，NAT模式，仅主机模式](https://blog.csdn.net/qq_39192827/article/details/85872025)

总结下来大致上如下？

<div style="overflow-x: auto;">
   <html lang="zh-CN">
   <head>
      <meta charset="UTF-8">
      <style>
         body {
               font-family: Arial, sans-serif;
               margin: 20px;
               line-height: 1.6;
         }
         h1 {
               color: #2c3e50;
               text-align: center;
               margin-bottom: 20px;
         }
         table {
               width: 100%;
               border-collapse: collapse;
               margin: 20px 0;
               box-shadow: 0 2px 5px rgba(0,0,0,0.1);
         }
         th, td {
               border: 1px solid #ddd;
               padding: 12px;
               text-align: left;
               white-space: nowrap; /* 所有文字不换行 */
         }
         th {
               background-color: #f2f2f2;
               font-weight: bold;
               color: #333;
         }
         tr:nth-child(even) {
               background-color: #f9f9f9;
         }
         tr:hover {
               background-color: #f1f1f1;
         }
         .note {
               font-style: italic;
               color: #666;
               margin-top: 20px;
               padding: 10px;
               background-color: #f8f8f8;
               border-left: 4px solid #2c3e50;
         }
      </style>
   </head>
   <body>
      <table>
         <thead>
               <tr>
                  <th>特性</th>
                  <th>仅主机模式 (Host-Only)</th>
                  <th>NAT 模式</th>
                  <th>桥接模式 (Bridged)</th>
               </tr>
         </thead>
         <tbody>
               <tr>
                  <td><strong>网络连接</strong></td>
                  <td>虚拟机与主机之间的私有网络</td>
                  <td>虚拟机通过主机访问外部网络</td>
                  <td>虚拟机直接连接到外部网络</td>
               </tr>
               <tr>
                  <td><strong>IP 地址</strong></td>
                  <td>虚拟机使用私有 IP 地址</td>
                  <td>虚拟机使用私有 IP 地址</td>
                  <td>虚拟机使用与主机相同的网络中的 IP 地址</td>
               </tr>
               <tr>
                  <td><strong>外部网络访问</strong></td>
                  <td>无法访问外部网络</td>
                  <td>可以访问外部网络</td>
                  <td>可以访问外部网络</td>
               </tr>
               <tr>
                  <td><strong>外部设备访问虚拟机</strong></td>
                  <td>外部设备无法访问虚拟机</td>
                  <td>外部设备无法直接访问虚拟机</td>
                  <td>外部设备可以直接访问虚拟机</td>
               </tr>
               <tr>
                  <td><strong>网络配置</strong></td>
                  <td>自动配置，无需手动干预</td>
                  <td>自动配置，无需手动干预</td>
                  <td>需要手动配置网络参数</td>
               </tr>
               <tr>
                  <td><strong>适用场景</strong></td>
                  <td>虚拟机与主机之间的通信</td>
                  <td>虚拟机需要访问互联网但不需要被外部访问</td>
                  <td>虚拟机需要与外部设备直接通信</td>
               </tr>
         </tbody>
      </table>
   </body>
   </html>
</div>