---
title: User-Defined Cloud
tags: 云计算论文研读
abbrlink: aeabe00a
date: 2021-07-15 11:20:51
---

Yiying Zhang, Ardalan Amiri Sani, and Guoqing Harry Xu. 2021. User-Defined Cloud. In Workshop on Hot Topics in Operating Systems (HotOS ’21), May 31–June 2, 2021, Ann Arbor, MI, USA. ACM, New York, NY, USA, 8 pages. https://doi.org/10.1145/3458336.3465304

<!-- more -->

## 引言

云使用者问题：

* 多花费35%的前在不需要的计算资源上。因为没有云服务能够匹配他们精确的需求。
* 部分领域的使用者不能在云上跑他们的负载。
* 安全问题。

云服务提供商问题：

* 新硬件部署和安全功能增加
* 兼容问题

原因就是云提供商定义和管理云去满足用户的需求。

主张用户定义云，云服务提供商创建和管理云。

User-Define Cloud(UDC)，用户可以定义硬件资源，执行环境，安全需要，系统功能。细粒度方式定义

## 优点

用户：

* 根据需要，自定义整个从软件到硬件的栈结构。
* 只需要支付他们需要的资源和功能
* 安全自定义

提供商：
* 独立的增加和删除软硬件功能。
* 更多的用户
* 更多的钱

## UDC提议

三个原则：

* Expressing definitions of low-level layers as runtime aspects.  
* Decouple specifications from their realization and decouple different aspects.
* Fine granularity at each layer.

