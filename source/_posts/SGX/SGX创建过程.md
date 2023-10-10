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
        <td> 允许 non-enclave 读 </td> <td> 允许 non-enclave 读 </td> <td> 允许 non-enclave 读 </td> <td> 允许 enclave 写 </td>
    </tr>
</table>

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
<ol>
<li>检查 DS:RBX 是否 32 字节对齐，DS:ECX 是否 4K 字节对齐，不是则 #GP(0)；</li>
<li>检查 DS:RCX 是否指向一个 EPC，不是则 #PF(DS:RCX)；</li>
<li>检查 DS:RBX.SRCPGE 是否 4K 字节对齐, DS:RBX.SECINFO 是否 64 字节对齐，不是则 #GP(0)；</li>
<li>检查 DS:RBX.LINADDR 是否为 0，不是则 #GP(0)；检查 DS:RBX.SECS 是否为 0，不是则 #GP(0)；</li>
<li>检查 DS:RBX.SECINFO 保留字段是否为 0，不是则 #GP(0)；检查 DS:RBX.SECINFO.FLAG.PT 是否为 PT_SECS，不是则 #GP(0)；</li>
<li>如果 DS:RCX 指向的 EPC 页面正在使用，且正在进行 VMX non-root 操作，且 ENABLE_EPC_VIRTUALIZATION_EXTENSIONS，则：
    

    VMCS.Exit_reason := SGX_CONFLICT; 
    VMCS.Exit_qualification.code := EPC_PAGE_CONFLICT_EXCEPTION;  
    VMCS.Exit_qualification.error := 0; 
    VMCS.Guest-physical_address := << translation of DS:RCX produced by paging >>;  
    VMCS.Guest-linear_address := DS:RCX;  
    Deliver VMEXIT;
    

</li>
<li>如果 DS:RCX 指向的 EPC 页面正在使用，是则 #GP(0)；</li>
<li>检查 EPCM(DS:RCX).VALID 是否为 1，是则 #PF(DS:RCX)；</li>
<li><font color=#FF0000> DS:RCX[32767:0] := DS:RBX.SRCPGE[32767:0]; </font> 注解：将 PAGEINFO 中 SRCPAGE 指向的页面拷贝到目标 SECS 中；</li>
<li>检查 XFRM 是否合法，且 DS:RCX.ATTRIBUTES.XFRM 最后两位为 3，不是则 #GP(0)；</li>
<li>检查 CET_ATTRIBUTES 是否合法，不是则 #GP(0)；判断伪代码如下（TMP_SECS 即 ECX）：

    
    IF ((DS:TMP_SECS.ATTRIBUTES.CET = 0 and DS:TMP_SECS.CET_ATTRIBUTES ≠ 0) ||
    (DS:TMP_SECS.ATTRIBUTES.CET = 0 and DS:TMP_SECS.CET_LEG_BITMAP_OFFSET ≠ 0) || 
    (CPUID.(EAX=7, ECX=0):EDX[CET_IBT] = 0 and DS:TMP_SECS.CET_LEG_BITMAP_OFFSET ≠ 0) || 
    (CPUID.(EAX=7, ECX=0):EDX[CET_IBT] = 0 and DS:TMP_SECS.CET_ATTRIBUTES[5:2] ≠ 0) || 
    (CPUID.(EAX=7, ECX=0):ECX[CET_SS] = 0 and DS:TMP_SECS.CET_ATTRIBUTES[1:0] ≠ 0) || 
    (DS:TMP_SECS.ATTRIBUTES.MODE64BIT = 1 and
    (DS:TMP_SECS.BASEADDR + DS:TMP_SECS.CET_LEG_BITMAP_OFFSET) not canonical) || 
    (DS:TMP_SECS.ATTRIBUTES.MODE64BIT = 0 and
    (DS:TMP_SECS.BASEADDR + DS:TMP_SECS.CET_LEG_BITMAP_OFFSET) & 0xFFFFFFFF00000000) || 
    (DS:TMP_SECS.CET_ATTRIBUTES.reserved fields not 0) or
    (DS:TMP_SECS.CET_LEG_BITMAP_OFFSET) is not page aligned)) 
    THEN
        #GP(0); 
    FI;
    

</li>
<li>通过 CPUID.(EAX=12H, ECX=0):EBX[31:0] 判断 DS:RCX.MISCSELECT[31:0] 中的功能是不是全部支持，不支持则 #GP(0)；</li>
<li>计算当发生 AEX 时，enclave 需要保存的状态大小 TMP_XSIZE，并检查 TMP_XSIZE 是否大于 <font color=#FF0000> DS:RCX.SSAFRAMESIZE*4096</font>，是则 #GP(0)；注解：DS:RCX.SSAFRAMESIZE 单位为 1 个页面；</li>
<li>检查 DS:RCX.BASEADDR 和 DS:RCX.SIZE 是否合法，不合法则 #GP(0)；<font color=#FF0000>注解：SIZE 至少两页面，且大小必须为 2 的幂，单位为字节；</font></li>
<li>检查 DS:RCX.ATTRIBUTES 中不包含 CR_SGX_ATTRIBUTES_MASK 不支持的属性；</li>
<li>检查 CONFIGID 和 CONFIGSVN 是否为 0 或 DS:TMP_SECS.ATTRIBUTES.KSS 不为 0，是则 #GP(0)；</li>
<li><font color=#FF0000>将 DS:RCX 设置为 Uninitialized，初始化 DS:RCX，计算 DS:RCX.MRENCLAVE；伪代码如下：</font>


    Clear DS:TMP_SECS to Uninitialized;
    DS:TMP_SECS.MRENCLAVE := SHA256INITIALIZE(DS:TMP_SECS.MRENCLAVE); 
    DS:TMP_SECS.ISVSVN := 0;
    DS:TMP_SECS.ISVPRODID := 0;
    (* Initialize hash updates etc*)
    Initialize enclave’s MRENCLAVE update counter;
    (* Add “ECREATE” string and SECS fields to MRENCLAVE *) 
    TMPUPDATEFIELD[63:0] := 0045544145524345H; // “ECREATE” 
    TMPUPDATEFIELD[95:64] := DS:TMP_SECS.SSAFRAMESIZE; 
    TMPUPDATEFIELD[159:96] := DS:TMP_SECS.SIZE;
    IF (CPUID.(EAX=7, ECX=0):EDX[CET_IBT] = 1) 
        THEN
            TMPUPDATEFIELD[223:160] := DS:TMP_SECS.CET_LEG_BITMAP_OFFSET; 
        ELSE
            TMPUPDATEFIELD[223:160] := 0;
    FI;
    TMPUPDATEFIELD[511:160] := 0;
    DS:TMP_SECS.MRENCLAVE := SHA256UPDATE(DS:TMP_SECS.MRENCLAVE, TMPUPDATEFIELD) 
    INC enclave’s MRENCLAVE update counter;
    (* Set EID *)
    DS:TMP_SECS.EID := LockedXAdd(CR_NEXT_EID, 1);
    (* Initialize the virtual child count to zero *) 
    DS:TMP_SECS.VIRTCHILDCNT := 0;
    (* Load ENCLAVECONTEXT with Address out of paging of SECS *)
    << store translation of DS:RCX produced by paging in SECS(DS:RCX).ENCLAVECONTEXT >>

    
</li>
<li><font color=#FF0000>更新 EPCM(DS:RCX)，注意 RWX 字段都为 0，说明 SECS 页面不能被应用直接读/写/执行；伪代码如下：</font>
    

    EPCM(DS:TMP_SECS).PT := PT_SECS;
    EPCM(DS:TMP_SECS).ENCLAVEADDRESS := 0;
    EPCM(DS:TMP_SECS).R := 0;
    EPCM(DS:TMP_SECS).W := 0; 
    EPCM(DS:TMP_SECS).X := 0;
    (* Set EPCM entry fields *) 
    EPCM(DS:RCX).BLOCKED := 0; 
    EPCM(DS:RCX).PENDING := 0; 
    EPCM(DS:RCX).MODIFIED := 0; 
    EPCM(DS:RCX).PR := 0; 
    EPCM(DS:RCX).VALID := 1;

    
</li>
</ol>

## EADD
### 简介
指令：ENCLS[ECREATE]

参数：EAX = 00H（输入）；RBX: PAGEINFO 地址（输入）；RCX: 目标 SECS 页面地址（输入）

当 SECS 被成功创建，enclave 页面通过 EADD 加入到 enclave。这包括将空闲的 EPC 页面转为 PT_REG 或 PT_TCS 类型。当调用 EADD 后，处理器会更新 EPCM 条目中的页面类型，enclave 访问页面的线性地址，页面的访问权限。它会将页面链接到输入的 SECS。EPCM 条目信息会被硬件利用去管理页面的访问控制。EADD 会记录 EPCM 的信息到 SECS 中存储的加密日志，并从 non-enclave 内存页面拷贝 4KB 数据到分配的 EPC 页面。

该函数从 non-enclave 内存页面拷贝数据到 EPC 页面，将 EPC 页面链接到 SECS 页面，最后存储线性地址和安全属性到 EPCM。enclave 偏移和安全属性会被计算测度并扩展到 SECS.MRENCLAVE。该指令只能在特权级为 0 时执行。


### 权限


### 并发限制

### 算法描述

## EEXTEND 

### 简介

### 权限


### 并发限制

### 算法描述

## EINIT 

### 简介

### 权限


### 并发限制

### 算法描述


