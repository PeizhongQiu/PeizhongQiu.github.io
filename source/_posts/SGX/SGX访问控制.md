---
title: SGX访问控制
abbrlink: 69e7e367
date: 2023-07-04 22:46:08
tags:
---

本文内容源自《Intel Software Developer’s Manual: Volumn 3》2023 年 3 月版本。

<!-- more -->

## 概述
每个 enclave 都有一段 ELRANGE 地址空间，该地址空间采用增强的访问控制。另外，ELRANGE 中的每个线性地址都必须映射到一个 EPC 页面，否则当 enclave 访问该线性地址时会产生错误。EPC 页面在物理上不必连续。系统软件负责分配 EPC 页面到各个 enclave 中。

Enclave 必须遵守 OS/VMM 的段页式策略。enclave 访问内存时，会使用已有的 IA32 段页式结构。OS/VMM 管理的页表和扩展页表为 enclave 页面提供地址转换。

进入 enclave 必须使用以下两条指令：ENCLU[EENTER]，ENCLU[ERESUME]；
退出 enclave 必须使用以下命令或事件：ENCLU[EEXIT]，AEX。

当 non-enclave 尝试访问映射到 EPC 页面的线性地址时，处理器会为了保护 enclave 的保密性和完整性做出一些行动，但是具体的行为在不同的执行中可能不同。比如说读取 enclave 页面可能返回一个值或者缓存行中的密文；写 enclave 页面可能会造成写丢弃。

## 术语

直接 enclave 访问（Direct EA）：enclave 内部执行的指令获取 ELRANGE 内存。
间接 enclave 访问（Indirect EA）：由特定的 SGX leaf function 访问 EPC 页面。
Enclave Access：Direct EA 和 Indirect Access 统称 EA。
Non-enclave Access：不是 EA 的内存访问。

## 访问控制

enclave 访问需要遵循以下访问控制：
1. 所有的内存访问都要遵循段页式保护措施；
2. 从 enclave 内部获取线性地址在 enclave 外部的代码会报 #GP(0)；
3. 从 enclave 内部对在 enclave 外部的线性地址使用 shadow-stack-load 和 shadow-stack-store 会报 #GP(0)；
4. 对 EPC 页面的 non-enclave 访问会造成未定义的错误；
5. 当使用 ENCLS[ADD]，ENCLS[EAUG] 分配 EPC 页面时，如果 EPC 的类型为 PT_REG，PT_TCS，PT_TRIM，则 EPC 页面必须映射到 ELRANGE 的指定位置。当 PFEC.SGX 位设置时，通过其他线性地址访问 enclave 会产生 #PF。
6. 直接 enclave 访问必须遵循 EPC 在 EPCM 中设置的安全属性。这些属性可以在 enclave 创建时定义，也可以使用 SGX2 相关指令修改。当 PFEC.SGX 位被设置时，检查失败会造成 #PF。直接 enclave 访问检查如下：
    - 目标页面必须属于当前执行的 enclave；
    - 如果 EPCM 中设置 EPC 页面有读/写权限，则 EPC 的数据可读/写；
    - 如果 EPCM 中设置 EPC 页面有执行权限，则可以从 EPC 页面中执行指令；
    - 只有当页面类型是 PT_SS_FIRST 和 PT_SS_REST 时，才能对 EPC 页面使用 Shadow-stack-load和 Shadow-stack-store；
    - 当页面类型是 PT_SS_FIRST 和 PT_SS_REST 时，不允许对 EPC 页面执行非 Shadow-stack-store 的写数据操作；
    - 目标页面不允许是 PT_SECS，PT_TCS，PT_VA，PT_TRIM 类型；
    - EPC 页面的状态不能为 BLOCKED，PENDING，MODIFIED。

## 基于段的访问控制

SGX 架构不会更改逻辑 CPU 执行的段检查。为了保证外面的实体不会以一种未知的行为修改逻辑地址到线性地址的映射，ENCLU[ENTER] 和 ENCLU[ERESUME] 会检查 CS，DS，ES 和 SS 的段基地址为 0。非 0 的段基地址会产生 #GP(0)。

在进入 enclave 时，处理器会保存 FS 和 GS 的内容，并读取 TCS 中存放的这些寄存器的值。后续就可以通过这些寄存器获取 TCS 中本地线程的数据。在退出 enclave 时，进入 enclave 时保存的内容会被恢复。

## 基于页的访问控制

当处理器在 enclave 内部执行时，内存数据访问可以跨 ELRANGE，即访问的数据部分在 ELRANGE，部分在 ELRANGE 外。此时，该访问会分成两次访问，一次是 ELRANGE 内访问，一次是 ELRANGE 外访问。内次访问会单独测评。但是获取代码不能跨 ELRANGE，会造成 #GP。

### 显式访问（Explicit Access）和隐式访问（Implicit Access）
显式访问：SGX 指令的参数有显式声明获取的内存地址或链接的数据结构。
显式访问一般使用逻辑地址。它们的获取受限于分段，分页，扩展分页等，可能会触发检查相关的错误/退出。

隐式访问：访问在处理器中缓存物理地址的数据结构。
这些地址不会在指令中当作参数传递，而是在指令使用时暗示。

这些访问不会触发访问相关的错误/退出/断点。下图列举了 SGX 指令的内存对象是显式访问还是隐式访问。显式访问对象的地址一般通过寄存器传递：RBX，RCX，RDX。
隐式访问用到的物理地址被不同的物理地址缓存，且存在不同的周期。当EPC 页面被 ENCLS[EADD] 或 ENCLS[EAUG] 加入到 enclave，或被 ENCLS[ELDB] 或 ENCLS[ELDU] 读取时，每个 EPC 页面关联的 SECS 的物理地址会被缓存。当 EPC 页面被 ENCLS[EREMOVE] 或 ENCLS[EWB] 删除时，相关的 SECS 缓存会被刷新。在进入 enclave 时，TCS 和 SSA 的物理地址会被缓存。在退出时，刷新缓存。

隐式访问的物理地址会在逻辑地址检查完后访问。在处理器缓存前，需要有数据结构的逻辑地址，这个地址会触发检查，但是在处理器缓存后，物理地址就不会触发检查。

{% asset_img 2.png 图1 SGX 指令的显式访问和隐式访问%}

{% asset_img 1.png 图2 SGX 指令的显式访问和隐式访问%}