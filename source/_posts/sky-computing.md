---
title: sky computing
tags: 云计算论文研读
abbrlink: ac1fc4f3
date: 2021-07-15 11:20:32
---

Ion Stoica and Scott Shenker. 2021. From Cloud Computing to Sky Computing. In Workshop on Hot Topics in Operating Systems (HotOS ’21), May 31-June 2, 2021, Ann Arbor, MI, USA. ACM, New York, NY, USA, 7 pages. https://doi.org/10.1145/3458336.3465302

<!-- more -->

## 引言

云中的服务都是有所有权的，而云服务提供商就是依靠提供这些具有所有权的服务相互区分。但是这些云服务提供商的竞争使得我们远离utility computing。

## 兼容层

兼容层把云提供的服务抽象出来，使得上层开发的应用能够在不同的云端上跑而且不需要修改。即应用可以直接调用兼容层的接口，然后应用就可以在任意云上跑。这可以类比网络上的IP层。但是，要更加广泛，更加难以定义，因为云给应用暴露了相当多的服务，因此会更像操作系统。

开源软件可以帮助我们更好的做好兼容层。问题是市场是否会支持这样的工作。

## 云间交互层

有了兼容层，用户还需要决定在哪片云上跑应用。网络上有BGP，sky computing应该有一个云间交互层将云服务提供商从用户中抽象出来，也就是说，用户不应该知道应用跑在那一片云上（除非用户想要明确知道）。

The intercloud layer must allow users to specify policies about where their jobs should run, but not require users to make low-level decisions about job placement (but would allow users to do so if they desired). These policies would allow a user to express their preferences about the tradeoff between performance, availability, and cost. In addition, a user might want to avoid their application running on a datacenter operated by a competitor, or stay within certain countries to obey relevant privacy regulations. To make this more precise, a user might specify that this is a Tensorflow job, it involves data that cannot leave Germany, and must be finished within the next two hours for under a certain cost.

问题与挑战：

* 服务命名模式。通过命名确定一个服务实例，元数据管理。
* 目录服务。寻找一个服务实例。维护元数据和命名信息。
* 账户和花费。

## 云间对等

对于大数据集，数据迁移不可避免。数据在不同云提供商间迁移是需要花费的，但是，这部分的花费可能比计算资源更便宜。互惠的数据对等管理会使数据的迁移更加快速有效。

## 未来

希望小的云服务提供商能够采用兼容层。

两种类型云服务提供商。

* stand-alone：专有的接口。更高的margin，专有的接口有更多可以创新提高的地方。
* commodity：使用sky computing。更低的margin，有更少可以创新提高的地方。每个云服务提供商可以专精一个方面。