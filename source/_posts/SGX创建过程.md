---
title: SGX创建过程
tags: SGX
abbrlink: ccbadbc2
date: 2023-07-04 14:55:02
---

本文内容源自《Intel Software Developer’s Manual: Volumn 3》2023 年 3 月版本。

<!-- more -->

enclave 创建过程主要包括以下步骤：<br>
1. 应用提交 enclave 内容和创建 enclave 的 API 需要的额外信息给运行在 <font color=#FF0000> 特权级为0 </font> 的 enclave 创建服务；
2. 运行在 0 特权级的 enclave 创建服务使用 ECREATE 叶函数去初始化环境，声明 enclave 基地址和大小。这个范围是应用地址空间的一部分，称为 ELRANGE。ECREATE 还为 SECS 分配 EPC 页面。<font color=#FF0000> 注意为 SECS 分配的 EPC 页面不必在 encalve 线性地址空间内，也不需要映射到进程</font>；
3. Enclave 创建服务使用 EADD 叶函数提交 EPC 页面到 enclave，使用 EEXTEND 计算 enclave 提交页面的测度。对于将要加入 enclave 的每个页面：使用 EADD 将新页面加入到 enclave；如果 enclave 开发人员需要页面的测度作为内容的证明，则使用 EEXTEND 计算页面 256 字节的测度，重复该步骤直到整个页面都被计算；
4. Enclave 创建服务使用 EINIT 叶函数完成 enclave 的创建过程，结束 enclave 测度计算并创建 enclave 实体。在 EINIT 执行之前，不允许执行 enclave 内部代码。

## ECREATE

### 简介

指令：ENCLS[ECREATE]

参数：EAX = 00H（输入）；RBX: PAGEINFO 地址（输入）；RCX: 目标 SECS 页面地址（输入）

ENCLS[ECREATE] 是 enclave 构建进程执行的第一条指令。ECREATE 拷贝外部的 SECS 到 EPC 页面中，之后内部的 SECS 不能被软件获取。

ECREATE 会初始化受保护的 SECS，并检查未使用的字段是否为0，并将其对应的 EPC 页面设置为 VALID 状态。 

应用提交的源 SECS 需要设置以下字段：BASEADDR，SIZE，ATTRIBUTES，CONFIGID，CONFIGSVN。SIZE 至少两个页面，BASEADDR 与 SIZE 边界对齐。

RBX 参数包含 PAGEINFO 的有效地址，PAGEINFO 包含源 SECS 的地址（SRCPAGE 字段）和 SECINFO 地址。PAGEINFO 的 SECS 字段未使用。SECINFO 字段必须指定 EPC 页面为 SECS 页面类型。

RCX 参数是目标 SECS 页面地址，该地址指向一个空白的 EPC 页面。

### 权限

<table>
    <tr>
        <th>PAGEINFO</th><th>PAGEINFO.SRCPGE</th><th>PAGEINFO.SECINFO</th><th>EPCPAGE</th>
    </tr>
    <tr>
        
    </tr>
</table>
PAGEINFO：允许 non-enclave 读

PAGEINFO.SRCPGE：允许 non-enclave 读

PAGEINFO.SECINFO：允许 non-enclave 读

EPCPAGE：允许 enclave 写

### 并发限制

<table>
    <tr>
        <th rowspan="2">Leaf function</th><th rowspan="2">Parameter</th><th colspan="3">Base Concurrency Restrictions</th>
    </tr>
    <tr>
        <th> Access</th><th>On Conflict</th><th>SGX_CONFLICT VM Exit Qualification</th>
    </tr>
    <tr>
        <td>ECREATE</td><td>SECS[DS:RCX]</td><td>Exclusive</td><td>#GP</td><td>EPC_PAGE_CONFLICT_EXCEPTION</td>
    </tr>
</table>

<table>
    <th rowspan="3">Leaf function</th><th rowspan="3">Parameter</th><th colspan="6">Additional Concurrency Restrictions</th>
    </tr>
    <tr>
        <th colspan="2">vs. EACCEPT, EACCEPTCOPY, EMODPE, EMODPR, EMODT </th><th colspan="2">vs. EADD, EEXTEND, EINIT</th><th colspan="2">vs. ETRACK, ETRACKC</th>
    </tr>
    <th> Access</th><th>On Conflict</th><th> Access</th><th>On Conflict</th><th> Access</th><th>On Conflict</th>
    <tr>
        <td>ECREATE</td><td>SECS[DS:RCX]</td><td>Concurrent</td><td></td><td>Concurrent</td><td></td><td>Concurrent</td><td></td>
    </tr>
<table>

### 算法描述





## EADD/EXTEND


## EINIT 