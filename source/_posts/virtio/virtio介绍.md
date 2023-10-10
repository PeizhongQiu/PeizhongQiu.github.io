---
title: virtio介绍
tags: virtio
abbrlink: 9e9d1546
date: 2023-07-15 16:19:40
---

本部分内容摘自《Viitual I/O Device(VIRTIO) Version 1.2》
<!-- more -->

本系列文档描述了“virtio”系列设备的规格。这些设备存在于虚拟环境中，但从设计上来说，对于虚拟机中的客户机来说，它们看起来就像物理设备，本文档也将它们视为物理设备。 这种相似性允许客户机使用标准驱动程序和发现机制。

virtio 和此规范的目的是虚拟环境和客户机应该为虚拟设备提供<font color=#FF0000>直接（Straightforward）、高效（Efficient）、标准（Standard）和可扩展（Extensible）</font>的机制，而不是针对每个环境或每个操作系统的精品机制。
 - 直接：Virtio 设备使用正常的中断和 DMA 总线机制，任何设备驱动程序作者都应该熟悉这些机制。没有奇特的翻页或 COW 机制：它只是一个普通的设备。
 - 高效：Virtio 设备由用于输入和输出的描述符环组成，这些描述符的布局整齐，以避免驱动程序和设备写入相同缓存行时产生缓存影响。
 - 标准：除了支持设备所连接的总线之外，Virtio 对其运行环境不做任何假设。在本规范中，virtio 设备是通过 MMIO、通道 I/O 和 PCI 总线传输实现的。
 - 可扩展：Virtio 设备包含在设备设置期间由客户机操作系统确认的功能位。 这允许向前和向后兼容：设备提供它所知道的所有功能，并且驱动程序承认它理解并希望使用的功能。

## 术语

<font color=#FF0000>MUST, REQUIRED, SHALL</font> mean that the definition is an absolute requirement of the specification.

<font color=#FF0000>MUST NOT, SHALL NOT</font> mean that the definition is an absolute prohibition of the specification.

<font color=#FF0000>SHOULD, RECOMMENDED</font> mean that there may exist valid reasons in particular circumstances to ignore a particular item, but the full implications must be understood and carefully weighed before choosing a different course.

<font color=#FF0000>SHOULD NOT, NOT RECOMMENDED</font> mean that there may exist valid reasons in particular circumstances when the particular behavior is acceptable or even useful, but the full implications should be understood and the case carefully weighed before implementing any behavior described with this label.

<font color=#FF0000>MAY, OPTIONAL</font> mean that an item is truly optional.  One vendor may choose to include the item because a particular marketplace requires it or because the vendor feels that it enhances the product while another vendor may omit the same item. An implementation which does not include a particular option MUST be prepared to interoperate with another implementation which does include the option, though perhaps with reduced functionality. In the same vein an implementation which does include a particular option MUST be prepared to interoperate with another implementation which does not include the option (except, of course, for the feature the option provides.)

### 旧版接口术语

本规范 1.0 版之前的规范草案定义了驱动程序和设备之间类似但不同的接口。由于这些已广泛部署，因此本规范包含 OPTIONAL 功能，以简化从这些早期草案接口的过渡。
具体来说，设备和驱动程序 MAY 支持：
- 旧版接口（Legacy Interface）：是本规范早期草案（1.0 之前）指定的接口；
- 旧版设备（Legacy Device）：是在该规范发布之前实现的设备，并在主机端实现遗留接口；
- 旧版驱动（Legacy Driver）：是在该规范发布之前实现的驱动程序，并在客户机端实现遗留接口；

旧版设备和旧版驱动程序不符合此规范。为了简化从这些早期草案接口的转换，设备 MAY 实现：
- 过渡设备（Transitional Device）：支持符合此规范的驱动程序并允许旧驱动程序的设备。

同样，驱动程序 MAY 实现：
- 过渡驱动（Transitional Driver）：支持符合此规范的设备和传统设备的驱动程序。

注意：旧接口不是必要的；即，除非需要向后兼容，否则不要实现它们！
不具有传统兼容性的设备或驱动程序分别称为非过渡设备和驱动程序。

## 结构规范
