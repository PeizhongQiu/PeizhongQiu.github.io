---
title: Unikraft
abbrlink: 8e98bdb9
date: 2023-10-08 23:22:09
tags:
---

## 什么是 Unikraft
<!-- more -->
参考文献：

## Unikraft 安装
源代码地址：
https://github.com/unikraft/unikraft

安装 kraft： 
```
curl --proto '=https' --tlsv1.2 -sSf https://get.kraftkit.sh | sh
```

参考网页：
[https://github.com/unikraft/kraftkit](https://github.com/unikraft/kraftkit)
https://unikraft.org/docs/getting-started

## Unikraft 调试
[https://www.youtube.com/watch?v=ANm0diPrWUE](https://www.youtube.com/watch?v=ANm0diPrWUE)
https://users.ece.utexas.edu/~adnan/gdb-refcard.pdf

qemu-system-x86_64  -s -S -kernel build/helloworld_qemu-x86_64 -nographic

gdb --eval-command="target remote :1234" build/helloworld_qemu-x86_64.dbg

## Unikraft qemu-x86-64启动流程
源码版本：0.13.1
### 寻找入口函数
方法 1：阅读 plat/kvm/Linker.uk：
```
ifeq (x86_64,$(CONFIG_UK_ARCH))
ifeq ($(CONFIG_KVM_BOOT_PROTO_MULTIBOOT),y)
KVM_LDFLAGS-y += -Wl,-m,elf_x86_64
KVM_LDFLAGS-y += -Wl,--entry=_multiboot_entry
ELF64_TO_32 = y
else ifeq ($(CONFIG_KVM_BOOT_PROTO_LXBOOT),y)
KVM_LDFLAGS-y += -Wl,--entry=_lxboot_entry
else ifeq ($(CONFIG_KVM_BOOT_PROTO_EFI_STUB),y)
KVM_LDFLAGS-y += -Wl,--entry=uk_efi_entry64
KVM_LDFLAGS-y += -Wl,-m,elf_x86_64
endif
```

从这里可以看出，当使用 multiboot 作为启动协议时，入口函数为 \_multiboot_entry。

方法 2：阅读 elf 文件。
以 helloworld 为例，使用以下命令：
```
readelf -a build/helloworld_qemu-x86_64.dbg
```
查看 Entry point address 选项：
```
Entry point address:               0x125f88
```
可知入口地址为 0x125f88。再进行反汇编：
```
objdump -ld -C -S build/helloworld_qemu-x86_64.dbg > helloworld.txt
```
查看 helloworld.txt 文件，并找到地址为 0x125f88 的函数：\_multiboot_entry。
### multiboot

https://www.gnu.org/software/grub/manual/multiboot/multiboot.html

### qemu-x86-64 启动源码阅读
由于使用 multiboot 启动，所以启动时的状态为：
- EAX包含一个魔数；
- EBX包含 multiboot 信息的32位物理地址；
- Flat 4GiB CS 和 DS 段，其中 ES、FS、GS 和 SS 设置为 DS；
- A20 已启用，启用保护模式，禁用分页，禁用中断。
注意：本博客暂不考虑重定向代码。
#### \_multiboot_entry
从 plat/kvm/x86/multiboot.S 的 \_multiboot_entry 开始阅读。
```
/* 与 multiboot 协议相关。*/
#define MULTIBOOT_FLAGS (MULTIBOOT_PAGE_ALIGN | MULTIBOOT_MEMORY_INFO)

.section .data.boot

.align 4
multiboot_header:
	.long MULTIBOOT_HEADER_MAGIC
	.long MULTIBOOT_FLAGS
	.long -(MULTIBOOT_HEADER_MAGIC+MULTIBOOT_FLAGS) /* checksum */

/**
* Stack and entry function to use during CPU initialization
*/
.section .bss
.space 4096 /* bss 段大小 */
lcpu_bootstack:

.section .rodata
lcpu_boot_startup_args:
	ur_data quad, multiboot_entry, 8 /* multiboot_entry 作为完成 lcpu_start64 后的初始化函数 */
	ur_data quad, lcpu_bootstack, 8

.code32
.section .text.32.boot
ENTRY(_multiboot_entry)

	/* only multiboot is supported for now */
	cmpl $MULTIBOOT_BOOTLOADER_MAGIC, %eax /* 判断是否是 multiboot 启动，不是则直接退出。*/
	jne no_multiboot

	/* Hardcoding for now I guess... */
	movl $0x00100000, %edi
	movl $0x00000000, %esi
	movl $0x00100000, %edx
	do_uk_reloc32 0
	
	/* startup args for boot CPU */
	ur_mov lcpu_boot_startup_args, %edi, 4, _phys /* 为后续完成 lcpu_start64 后跳转做准备 */
	movl %ebx, %esi /* multiboot info */
	ur_mov lcpu_start32, %ebx, 4, _phys
	jmp *%ebx /* 执行 lcpu_start32 */

no_multiboot:
	cli
1:
	hlt
	jmp 1b
END(_multiboot_entry)
```
注意在执行完  \_multiboot_entry 之后，esi 里存储的是包含 multiboot 信息的32位物理地址，edi 里存储的是 lcpu_boot_startup_args 的地址，ebx 里存储的是 lcpu_start32 的函数地址。所以函数会跳转到 lcpu_start32。

#### lcpu_start32
lcpu_start32 打开 PAE，IA-32e，设置页表，cr0，GDT，将程序跳转到 64 位环境执行。
```
.code32
.section .text.boot.32
ENTRY(lcpu_start32) /* 打开 PAE，IA-32e，设置页表，cr0，GDT */
	/* Enable physical address extension (PAE) */
	movl $CR4_BOOT32_SETTINGS, %eax
	movl %eax, %cr4
	
	/* Switch to IA-32e mode (long mode) */ /* IA-32e 是 64 位*/
	xorl %edx, %edx
	movl $EFER_BOOT32_SETTINGS, %eax
	movl $X86_MSR_EFER, %ecx
	wrmsr
	/* 页表采用的是 4 级分页，需要提前设置 CR0.PG，CR4.PAE，IA32_EFER.LME 为 1，CR4.LA57 为 0.*/
	/* Set boot page table and enable paging */ /* x86_bpt_pml4 里面是页表 */
	ur_mov x86_bpt_pml4, %eax, 4, _phys
	movl %eax, %cr3
	
	/* Protected mode, Write Protect, Paging */
	movl $CR0_BOOT32_SETTINGS, %eax
	movl %eax, %cr0

	/* Load 64-bit GDT and jump to 64-bit code segment */
	ur_mov gdt64_ptr, %eax, 4
	lgdt (%eax)

	/* Again, we use the same strategy, only this time we generate an actual
	* uk_reloc entry to be automatically resolved by the early relocator,
	* instead of relying on the code that relocates the start16 section
	* before starting the Application Processors, since execution of
	* lcpu_start32 comes before that.
	*/
	ur_mov jump_to64, %eax, 4
	movl %eax, -6(%eax)
	ljmp $(gdt64_cs - gdt64), $START32_PLACEHOLDER
.code64
jump_to64:
	/* Set up remaining segment registers */
	movl $(gdt64_ds - gdt64), %eax
	movl %eax, %es
	movl %eax, %ss
	movl %eax, %ds
	xorl %eax, %eax
	movl %eax, %fs
	movl %eax, %gs
	leaq lcpu_start64(%rip), %rcx
	jmp *%rcx
END(lcpu_start32)
```
分析页表 x86_bpt_pml4。
首先需要了解 64 位内存分页。该部分采用的是 4 级分页。转换过程参考：https://zhuanlan.zhihu.com/p/652983618。

通过阅读 x86_bpt_pml4 源码，可以得到页表的示意图如下图所示。
{% asset_img  页表.jpg 图1 页表%}

#### lcpu_start64

```
.code64
.section .text.boot.64
ENTRY(lcpu_start64) /* 将 lcpu 状态设为 init，设置栈，打开 FPU，SSE，XSAVE，AVX，FS，GS，PKU，跳转到 multiboot_entry */
	/* Save the startup args pointer */
	movq	%rdi, %r8	/* r8 存储的是 lcpu_boot_startup_args 地址。*/

	/* Request basic CPU features and APIC ID
	 * TODO: This APIC ID is limited to 256. Better get from leaf 0x1f
	 */
	movl	$1, %eax
	cpuid
	shrl	$24, %ebx

	/* Use APIC_ID * LCPU_SIZE for indexing the cpu structure */
	movl	$LCPU_SIZE, %eax
	imul	%ebx, %eax

	/* Compute pointer into CPU struct array and store it in RBP
	 * We do not use the frame pointer, yet
	 *//* lcpus 为数组，用来存储所有逻辑 cpu 的元数据。*/
	leaq	lcpus(%rip), %rbp
	addq	%rax, %rbp

	/* Put CPU into init state */ /* lcpu 状态设为 init 状态*/
	movl	$LCPU_STATE_INIT, LCPU_STATE_OFFSET(%rbp)

	/* Enable FPU, SSE, (XSAVE feature), (AVX), FS and GS base, (memory protection keys (PKU)). Request extended CPU features.()表示需要配置相应启动参数。*/
	......

	/* Check if we have startup arguments supplied */
	test	%r8, %r8
	jz	no_args

	/* Initialize the CPU configuration with the supplied startup args */
	movq	LCPU_SARGS_ENTRY_OFFSET(%r8), %rax	/* multiboot_entry 函数地址 */
	movq	LCPU_SARGS_STACKP_OFFSET(%r8), %rsp /*bss 栈地址*/

	jmp	jump_to_entry

no_args:
	/* Load the stack pointer and the entry address from the CPU struct */
	movq	LCPU_ENTRY_OFFSET(%rbp), %rax
	movq	LCPU_STACKP_OFFSET(%rbp), %rsp

jump_to_entry:
	/* According to System V AMD64 the stack pointer must be aligned to
	 * 16-bytes. In other words, the value (RSP+8) must be a multiple of
	 * 16 when control is transferred to the function entry point (i.e.,
	 * the compiler expects a misalignment due to the return address having
	 * been pushed onto the stack).
	 */
	andq	$~0xf, %rsp
	subq	$0x8, %rsp

	movq	%rbp, %rdi	/*rdi 为第一个参数，即 rbp，rbp 存储的为该逻辑 cpu 在 lcpus 数组中的地址*/
#if !__OMIT_FRAMEPOINTER__
	/* Reset frame pointer */
	xorq	%rbp, %rbp
#endif /* !__OMIT_FRAMEPOINTER__ */

	/* Arguments for entry function
	 * arg0 @ RDI = this CPU, arg1 @ RSI = boot parameters (if available)
	 */
	jmp	*%rax	/* 转移到 multiboot_entry*/
	/* rsi 在 multiboot.S 中赋值*/
fail:
	movl	$LCPU_STATE_HALTED, LCPU_STATE_OFFSET(%rbp)

fail_loop:
	cli
1:
	hlt
	jmp	1b
END(lcpu_start64)
```

执行完 lcpu_start64，转向 multiboot_entry 执行。

#### multiboot_entry
当执行 hello-world 程序时，参数 mi 的值：
```
(gdb) p/x *mi
$1 = {flags = 0x24f, mem_lower = 0x27f, mem_upper = 0x1fb80, boot_device = 0x8000ffff, cmdline = 0x185000, mods_count = 0x0, mods_addr = 0x185000,
  u = {aout_sym = {tabsize = 0x0, strsize = 0x0, addr = 0x0, reserved = 0x0}, elf_sec = {num = 0x0, size = 0x0, addr = 0x0, shndx = 0x0}},
  mmap_length = 0x90, mmap_addr = 0x9000, drives_length = 0x0, drives_addr = 0x0, config_table = 0x0, boot_loader_name = 0x18501e, apm_table = 0x0,
  vbe_control_info = 0x0, vbe_mode_info = 0x0, vbe_mode = 0x0, vbe_interface_seg = 0x0, vbe_interface_off = 0x0, vbe_interface_len = 0x0,
  framebuffer_addr = 0x0, framebuffer_pitch = 0x0, framebuffer_width = 0x0, framebuffer_height = 0x0, framebuffer_bpp = 0x0, framebuffer_type = 0x0, {{
      framebuffer_palette_addr = 0x0, framebuffer_palette_num_colors = 0x0}, {framebuffer_red_field_position = 0x0, framebuffer_red_mask_size = 0x0,
      framebuffer_green_field_position = 0x0, framebuffer_green_mask_size = 0x0, framebuffer_blue_field_position = 0x0,
      framebuffer_blue_mask_size = 0x0}}}

(gdb) p (char*)mi->cmdline
$2 = 0x185000 "build/helloworld_qemu-x86_64 "

(gdb) p (char*)mi->boot_loader_name
$3 = 0x18501e "qemu"
```
参数 lcpu 值为：
```
(gdb) p/x *lcpu
$4 = {state = 0x1, idx = 0x0, id = 0x0, {s_args = {entry = 0x0, stackp = 0x0}, error_code = 0x0}, arch = {<No data fields>}}
```

下面阅读 multiboot_entry 函数，调试信息为执行 hello-world 程序的信息。

```
void multiboot_entry(struct lcpu *lcpu, struct multiboot_info *mi)
{
	struct ukplat_bootinfo *bi;
	struct ukplat_memregion_desc mrd = {0};
	multiboot_memory_map_t *m;
	multiboot_module_t *mods;
	__sz offset, cmdline_len;
	__paddr_t start, end;
	__u32 i;
	int rc;
	/* uk_boootinfo 节内容通过 support/scripts/mkbootinfo.py（91-101行）完成初始化。*/
	bi = ukplat_bootinfo_get();
	/**
	(gdb) p/x *bi
	$1 = {magic = 0xb007b0b0, version = 0x1, _pad0 = {0x0, 0x0, 0x0}, bootloader = {0x0 <repeats 16 times>}, bootprotocol = {
    0x0 <repeats 16 times>}, cmdline = 0x0, cmdline_len = 0x0, dtb = 0x0, efi_st = 0x0, mrds = {capacity = 0x80, count = 0x4,
    mrds = 0x12f2e8 <bi_bootinfo_sec+80>}}
	(gdb) p/x *(bi->mrds.mrds)@4
	$2 = {{pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {pbase = 0x127000, vbase = 0x127000,
    len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000, type = 0x4, flags = 0x3}, {
    pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}}
	*/
	if (unlikely(!bi))
		multiboot_crash("Incompatible or corrupted bootinfo", -EINVAL);

	/* We have to call this here as the very early do_uk_reloc32 relocator
	 * does not also relocate the UKPLAT_MEMRT_KERNEL mrd's like its C
	 * equivalent, do_uk_reloc, does.
	 */
	do_uk_reloc_kmrds(0, 0);

	/* Ensure that the memory map contains the legacy high mem area *//* bi->mrds 中插入 HI_MEM 和 BIOS_ROM。*/
	rc = ukplat_memregion_list_insert_legacy_hi_mem(&bi->mrds);
	/*
  	(gdb) p/x bi->mrds
	$3 = {capacity = 0x80, count = 0x6, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@6
	$4 = {{pbase = 0xa0000, vbase = 0xa0000, len = 0x40000, type = 0x2, flags = 0x23}, {pbase = 0xe0000, vbase = 0xe0000,
      len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {
      pbase = 0x127000, vbase = 0x127000, len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000,
      type = 0x4, flags = 0x3}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}}
	*/
	if (unlikely(rc))
		multiboot_crash("Could not insert legacy memory region", rc);

	/* Add the cmdline */ /* 将 mi->cmdline 相关信息存到 bi->mrds，bi->cmdline 和 bi->cmdline_len。*/
	if (mi->flags & MULTIBOOT_INFO_CMDLINE) {
		if (mi->cmdline) { /* 当使用 hellowaorld 时，mi->cmdline 为 build/helloworld_qemu-x86_64 */
			cmdline_len = strlen((const char *)(__uptr)mi->cmdline);
			mrd.pbase = mi->cmdline;
			mrd.vbase = mi->cmdline; /* 1:1 mapping */
			mrd.len   = cmdline_len;
			mrd.type  = UKPLAT_MEMRT_CMDLINE;
			mrd.flags = UKPLAT_MEMRF_READ | UKPLAT_MEMRF_MAP;

			mrd_insert(bi, &mrd);

			bi->cmdline = mi->cmdline;
			bi->cmdline_len = cmdline_len;
		}
	}
	/*
	(gdb) p (char *)(bi->cmdline)
	$8 = 0x185000 "build/helloworld_qemu-x86_64 "
	(gdb) p/x (bi->cmdline_len)
	$9 = 0x1d
	(gdb) p/x bi->mrds
	$10 = {capacity = 0x80, count = 0x7, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@7
	$11 = {{pbase = 0xa0000, vbase = 0xa0000, len = 0x40000, type = 0x2, flags = 0x23}, {pbase = 0xe0000, vbase = 0xe0000,
    	len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {
    	pbase = 0x127000, vbase = 0x127000, len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000,
    	type = 0x4, flags = 0x3}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}, {pbase = 0x185000,
    	vbase = 0x185000, len = 0x1d, type = 0x10, flags = 0x21}}
	*/

	/* Copy boot loader */ /*更新 bi->bootloader，bi->bootprotocol。*/
	if (mi->flags & MULTIBOOT_INFO_BOOT_LOADER_NAME) {
		if (mi->boot_loader_name) { /* 当使用 build/helloworld_qemu-x86_64 时，mi->boot_loader_name 为 qemu。*/
			strncpy(bi->bootloader,
				(const char *)(__uptr)mi->boot_loader_name,
				sizeof(bi->bootloader) - 1);
		}
	}
	/*
	(gdb) p (char*)(bi->bootloader)
	$12 = 0x12f2a0 <bi_bootinfo_sec+8> "qemu"
	*/
	memcpy(bi->bootprotocol, "multiboot", sizeof("multiboot"));
	/*
	(gdb) p (char*)(bi->bootprotocol)
	$14 = 0x12f2b0 <bi_bootinfo_sec+24> "multiboot"
	*/

	/* Add modules from the multiboot info to the memory region list */ /* 将 mod 相关信息加载到 bi->mrds。*/
	if (mi->flags & MULTIBOOT_INFO_MODS) {
		mods = (multiboot_module_t *)(__uptr)mi->mods_addr;
		for (i = 0; i < mi->mods_count; i++) {
			mrd.pbase = mods[i].mod_start;
			mrd.vbase = mods[i].mod_start; /* 1:1 mapping */
			mrd.len   = mods[i].mod_end - mods[i].mod_start;
			mrd.type  = UKPLAT_MEMRT_INITRD;
			mrd.flags = UKPLAT_MEMRF_READ | UKPLAT_MEMRF_MAP;

#ifdef CONFIG_UKPLAT_MEMRNAME
			strncpy(mrd.name, (char *)(__uptr)mods[i].cmdline,
				sizeof(mrd.name) - 1);
#endif /* CONFIG_UKPLAT_MEMRNAME */

			mrd_insert(bi, &mrd);
		}
	}

#ifdef CONFIG_UKPLAT_MEMRNAME
	memset(mrd.name, 0, sizeof(mrd.name));
#endif /* CONFIG_UKPLAT_MEMRNAME */

	/* Add map ranges from the multiboot info to the memory region list
	 * CAUTION: These could generally overlap with regions already in the
	 * list. We thus split new free regions accordingly to remove allocated
	 * ranges. For all other ranges, we assume RESERVED type and that they
	 * do NOT overlap with other allocated ranges (e.g., modules).
	 */ /*将 multiboot info 映射加入到 bi->mrds。multiboot info 信息从第二页开始。*/
	/*
	(gdb) p/x *(multiboot_memory_map_t*)(mi->mmap_addr)@6
	$20 = {{size = 0x14, addr = 0x0, len = 0x9fc00, type = 0x1}, {size = 0x14, addr = 0x9fc00, len = 0x400, type = 0x2}, {size = 0x14,
    addr = 0xf0000, len = 0x10000, type = 0x2}, {size = 0x14, addr = 0x100000, len = 0x7ee0000, type = 0x1}, {size = 0x14,
    addr = 0x7fe0000, len = 0x20000, type = 0x2}, {size = 0x14, addr = 0xfffc0000, len = 0x40000, type = 0x2}}
	*/
	if (mi->flags & MULTIBOOT_INFO_MEM_MAP) {
		for (offset = 0; offset < mi->mmap_length;
		     offset += m->size + sizeof(m->size)) {
			m = (void *)(__uptr)(mi->mmap_addr + offset);

			start = MAX(m->addr, __PAGE_SIZE);
			end   = m->addr + m->len;
			if (unlikely(end <= start || end - start < PAGE_SIZE))
				continue;

			mrd.pbase = start;
			mrd.vbase = start; /* 1:1 mapping */
			mrd.len   = end - start;

			if (m->type == MULTIBOOT_MEMORY_AVAILABLE) {
				mrd.type  = UKPLAT_MEMRT_FREE;
				mrd.flags = UKPLAT_MEMRF_READ |
					    UKPLAT_MEMRF_WRITE;
			} else {
				mrd.type  = UKPLAT_MEMRT_RESERVED;
				mrd.flags = UKPLAT_MEMRF_READ |
					    UKPLAT_MEMRF_MAP;

				/* We assume that reserved regions cannot
				 * overlap with loaded modules.
				 */
			}

			mrd_insert(bi, &mrd);
		}
	}
	/*
	(gdb) p/x bi->mrds
	$22 = {capacity = 0x80, count = 0xc, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@12
	$24 = {{pbase = 0x1000, vbase = 0x1000, len = 0x9ec00, type = 0x1, flags = 0x3}, {pbase = 0xa0000, vbase = 0xa0000, len = 0x40000,
    type = 0x2, flags = 0x23}, {pbase = 0xf0000, vbase = 0xf0000, len = 0x10000, type = 0x2, flags = 0x21}, {pbase = 0xe0000,
    vbase = 0xe0000, len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0x100000, vbase = 0x100000, len = 0x7ee0000, type = 0x1,
    flags = 0x3}, {pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {pbase = 0x127000,
    vbase = 0x127000, len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000, type = 0x4,
    flags = 0x3}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}, {pbase = 0x185000, vbase = 0x185000,
    len = 0x1d, type = 0x10, flags = 0x21}, {pbase = 0x7fe0000, vbase = 0x7fe0000, len = 0x20000, type = 0x2, flags = 0x21}, {
    pbase = 0xfffc0000, vbase = 0xfffc0000, len = 0x40000, type = 0x2, flags = 0x21}}
	*/
	/* 对 bi->mrds 进行整理，合并*/
	rc = ukplat_memregion_list_coalesce(&bi->mrds);
	/*
	(gdb) p/x bi->mrds
	$25 = {capacity = 0x80, count = 0xb, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@11
	$26 = {{pbase = 0x1000, vbase = 0x1000, len = 0x9ec00, type = 0x1, flags = 0x3}, {pbase = 0xa0000, vbase = 0xa0000, len = 0x40000,
    type = 0x2, flags = 0x23}, {pbase = 0xe0000, vbase = 0xe0000, len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0x100000,
    vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {pbase = 0x127000, vbase = 0x127000, len = 0x8000, type = 0x4,
    flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000, type = 0x4, flags = 0x3}, {pbase = 0x12f000,
    vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}, {pbase = 0x185000, vbase = 0x185000, len = 0x1d, type = 0x10,
    flags = 0x21}, {pbase = 0x18501d, vbase = 0x18501d, len = 0x7e5afe3, type = 0x1, flags = 0x3}, {pbase = 0x7fe0000,
    vbase = 0x7fe0000, len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0xfffc0000, vbase = 0xfffc0000, len = 0x40000,
    type = 0x2, flags = 0x21}}
	*/
	if (unlikely(rc))
		multiboot_crash("Could not coalesce memory regions", rc);

	rc = ukplat_memregion_alloc_sipi_vect();
	if (unlikely(rc))
		multiboot_crash("Could not insert SIPI vector region", rc);

	_ukplat_entry(lcpu, bi);
}
```
综上所述，multiboot_entry 完成了 bi 初始化，并转到 _ukplat_entry 执行。

#### _ukplat_entry
下面阅读 _ukplat_entry 函数，调试信息为执行 hello-world 程序的信息。

```
void _ukplat_entry(struct lcpu *lcpu, struct ukplat_bootinfo *bi)
{
	int rc;
	void *bstack;

	_libkvmplat_init_console();

	/* Initialize trap vector table */
	traps_table_init();

	/* Initialize LCPU of bootstrap processor */
	rc = lcpu_init(lcpu);
	/*
	(gdb) p/x *lcpu
	$1 = {state = 0x3, idx = 0x0, id = 0x0, {s_args = {entry = 0x0, stackp = 0x0}, error_code = 0x0}, arch = {<No data fields>}}
	*/
	if (unlikely(rc))
		UK_CRASH("Bootstrap processor init failed: %d\n", rc);

	/* Initialize IRQ controller */
	intctrl_init();

	/* Initialize command line *//* 将 bi->cmdline 加入到 bi->mrds*/
	rc = cmdline_init(bi);
	/*
	(gdb) p/x bi->mrds
	$2 = {capacity = 0x80, count = 0xc, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@12
	$3 = {{pbase = 0x1000, vbase = 0x1000, len = 0x1000, type = 0x4, flags = 0x23}, {pbase = 0x2000, vbase = 0x2000, len = 0x9dc00,
    type = 0x1, flags = 0x3}, {pbase = 0xa0000, vbase = 0xa0000, len = 0x40000, type = 0x2, flags = 0x23}, {pbase = 0xe0000,
    vbase = 0xe0000, len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4,
    flags = 0x5}, {pbase = 0x127000, vbase = 0x127000, len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000,
    len = 0x56000, type = 0x4, flags = 0x3}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}, {
    pbase = 0x185000, vbase = 0x185000, len = 0x1d, type = 0x10, flags = 0x21}, {pbase = 0x18501d, vbase = 0x18501d,
    len = 0x7e5afe3, type = 0x1, flags = 0x3}, {pbase = 0x7fe0000, vbase = 0x7fe0000, len = 0x20000, type = 0x2, flags = 0x21}, {
    pbase = 0xfffc0000, vbase = 0xfffc0000, len = 0x40000, type = 0x2, flags = 0x21}}
	*/
	if (unlikely(rc))
		UK_CRASH("Cmdline init failed: %d\n", rc);

	/* Allocate boot stack */
	bstack = ukplat_memregion_alloc(__STACK_SIZE, UKPLAT_MEMRT_STACK,
					UKPLAT_MEMRF_READ |
					UKPLAT_MEMRF_WRITE |
					UKPLAT_MEMRF_MAP);
	/*
	(gdb) p/x bi->mrds
	$4 = {capacity = 0x80, count = 0xd, mrds = 0x12f2e8 <bi_bootinfo_sec+80>}
	(gdb) p/x *(bi->mrds.mrds)@13
	$5 = {{pbase = 0x1000, vbase = 0x1000, len = 0x1000, type = 0x4, flags = 0x23}, {pbase = 0x2000, vbase = 0x2000, len = 0x10000,
    type = 0x40, flags = 0x23}, {pbase = 0x12000, vbase = 0x12000, len = 0x8dc00, type = 0x1, flags = 0x3}, {pbase = 0xa0000,
    vbase = 0xa0000, len = 0x40000, type = 0x2, flags = 0x23}, {pbase = 0xe0000, vbase = 0xe0000, len = 0x20000, type = 0x2,
    flags = 0x21}, {pbase = 0x100000, vbase = 0x100000, len = 0x27000, type = 0x4, flags = 0x5}, {pbase = 0x127000,
    vbase = 0x127000, len = 0x8000, type = 0x4, flags = 0x1}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x56000, type = 0x4,
    flags = 0x3}, {pbase = 0x12f000, vbase = 0x12f000, len = 0x0, type = 0x4, flags = 0x3}, {pbase = 0x185000, vbase = 0x185000,
    len = 0x1d, type = 0x10, flags = 0x21}, {pbase = 0x18501d, vbase = 0x18501d, len = 0x7e5afe3, type = 0x1, flags = 0x3}, {
    pbase = 0x7fe0000, vbase = 0x7fe0000, len = 0x20000, type = 0x2, flags = 0x21}, {pbase = 0xfffc0000, vbase = 0xfffc0000,
    len = 0x40000, type = 0x2, flags = 0x21}}
	*/
	if (unlikely(!bstack))
		UK_CRASH("Boot stack alloc failed\n");

	bstack = (void *)((__uptr)bstack + __STACK_SIZE);

	/* Initialize memory */ /* 保证 bi->mrds 中的数据全部都在 bpt_unmap_mrd 的映射范围内。*/
	rc = ukplat_mem_init();
	if (unlikely(rc))
		UK_CRASH("Mem init failed: %d\n", rc);

	/* Print boot information */
	ukplat_bootinfo_print();

#if defined(CONFIG_HAVE_SMP) && defined(CONFIG_UKPLAT_ACPI)
	rc = acpi_init();
	if (likely(rc == 0)) {
		rc = lcpu_mp_init(CONFIG_UKPLAT_LCPU_RUN_IRQ,
				  CONFIG_UKPLAT_LCPU_WAKEUP_IRQ,
				  NULL);
		if (unlikely(rc))
			uk_pr_err("SMP init failed: %d\n", rc);
	} else {
		uk_pr_err("ACPI init failed: %d\n", rc);
	}
#endif /* CONFIG_HAVE_SMP && CONFIG_UKPLAT_ACPI */

#ifdef CONFIG_HAVE_SYSCALL
	_init_syscall();
#endif /* CONFIG_HAVE_SYSCALL */

#if CONFIG_HAVE_X86PKU
	_check_ospke();
#endif /* CONFIG_HAVE_X86PKU */

	/* Switch away from the bootstrap stack */
	uk_pr_info("Switch from bootstrap stack to stack @%p\n", bstack);
	/*更换栈为 bstack，并执行 _ukplat_entry2。*/
	lcpu_arch_jump_to(bstack, _ukplat_entry2);
}

```

```
static void __noreturn _ukplat_entry2(void)
{
	/* It's not possible to unwind past this function, because the stack
	 * pointer was overwritten in lcpu_arch_jump_to. Therefore, mark the
	 * previous instruction pointer as undefined, so that debuggers or
	 * profilers stop unwinding here.
	 */
	ukarch_cfi_unwind_end();

	ukplat_entry_argp(NULL, cmdline, cmdline_len);

	ukplat_lcpu_halt();
}
```

_ukplat_entry2 转到 ukplat_entry_argp 运行。

#### ukplat_entry_argp 和 ukplat_entry 

```
/* defined in <uk/plat.h> */
void ukplat_entry_argp(char *arg0, char *argb, __sz argb_len)
{
	static char *argv[CONFIG_LIBUKBOOT_MAXNBARGS];
	int argc = 0;

	if (arg0) {
		argv[0] = arg0;
		argc += 1;
	}
	if (argb && argb_len) {
		argc += uk_argnparse(argb, argb_len, arg0 ? &argv[1] : &argv[0],
				     arg0 ? (CONFIG_LIBUKBOOT_MAXNBARGS - 1)
					  : CONFIG_LIBUKBOOT_MAXNBARGS);
	}
	ukplat_entry(argc, argv);
}
```

转到 ukplat_entry 运行。

```
void ukplat_entry(int argc, char *argv[])
{
#if CONFIG_LIBPOSIX_ENVIRON
	char **envp;
#endif /* CONFIG_LIBPOSIX_ENVIRON */
	int rc = 0;
#if CONFIG_LIBUKALLOC
	struct uk_alloc *a = NULL;
#endif
#if !CONFIG_LIBUKBOOT_NOALLOC
	void *tls = NULL;
#endif
#if CONFIG_LIBUKSCHED
	struct uk_sched *s = NULL;
#endif
	uk_ctor_func_t *ctorfn;
	uk_init_func_t *initfn;
	int i;

	uk_pr_info("Unikraft constructor table at %p - %p\n",
		   &uk_ctortab_start[0], &uk_ctortab_end);
	/*
	* UK_CTOR_PRIO(_init_ectx_store, 0);
	* static struct pci_bus_handler ph = {
	*	.b.init = pci_init,
	*	.b.probe = pci_probe,
	*	.drv_list = UK_LIST_HEAD_INIT(ph.drv_list),
	*	.dev_list = UK_LIST_HEAD_INIT(ph.dev_list),
	* };
	* UK_BUS_REGISTER(&ph.b);
	* UK_BUS_REGISTER(&virtio_bus);
	*//* 将 pci 和 virtio 添加到总线。*/
	uk_ctortab_foreach(ctorfn, uk_ctortab_start, uk_ctortab_end) {
		UK_ASSERT(*ctorfn);
		uk_pr_debug("Call constructor: %p())...\n", *ctorfn);
		(*ctorfn)();
	}

	......

#if !CONFIG_LIBUKBOOT_NOALLOC
	uk_pr_info("Initialize memory allocator...\n");
	/* 初始化内存分配器。并将 bi->mrds 中的内存区域加入到内存分配中管理。*/
	a = heap_init();
	if (unlikely(!a))
		UK_CRASH("Failed to initialize memory allocator\n");
	else {
		/* 将 a 设置为默认内存分配器。*/
		rc = ukplat_memallocator_set(a);
		if (unlikely(rc != 0))
			UK_CRASH("Could not set the platform memory allocator\n");
	}

	/* Allocate a TLS for this execution context */
	tls = uk_memalign(a,
			  ukarch_tls_area_align(),
			  ukarch_tls_area_size());
	if (!tls)
		UK_CRASH("Failed to allocate and initialize TLS\n");

	/* Copy from TLS master template */
	ukarch_tls_area_init(tls);
	/* Activate TLS */
	ukplat_tlsp_set(ukarch_tls_tlsp(tls));
#endif /* !CONFIG_LIBUKBOOT_NOALLOC */

#if CONFIG_LIBUKALLOC
	uk_pr_info("Initialize IRQ subsystem...\n");
	/* 关闭 IRQ。*/
	rc = ukplat_irq_init(a);
	if (unlikely(rc != 0))
		UK_CRASH("Could not initialize the platform IRQ subsystem\n");
#endif

	/* On most platforms the timer depend on an initialized IRQ subsystem */
	uk_pr_info("Initialize platform time...\n");
	ukplat_time_init();

#if !CONFIG_LIBUKBOOT_NOSCHED
	uk_pr_info("Initialize scheduling...\n");
#if CONFIG_LIBUKBOOT_INITSCHEDCOOP
	s = uk_schedcoop_create(a);/*创建 idle 线程。*/
#endif
	if (unlikely(!s))
		UK_CRASH("Failed to initialize scheduling\n");
	uk_sched_start(s); /* 创建 init 线程。*/
#endif /* !CONFIG_LIBUKBOOT_NOSCHED */

	/* Enable interrupts before starting the application */
	ukplat_lcpu_enable_irq();

	/**
	 * Run init table
	 */
	/** init.h 中定义了相关宏。
	 * uk_initcall_class_prio(uk_bus_lib_init, UK_BUS_INIT_CLASS, UK_BUS_INIT_PRIO);
	*/
	uk_pr_info("Init Table @ %p - %p\n",
		   &uk_inittab_start[0], &uk_inittab_end);
	uk_inittab_foreach(initfn, uk_inittab_start, uk_inittab_end) {
		UK_ASSERT(*initfn);
		uk_pr_debug("Call init function: %p()...\n", *initfn);
		rc = (*initfn)();
		if (rc < 0) {
			uk_pr_err("Init function at %p returned error %d\n",
				  *initfn, rc);
			rc = UKPLAT_CRASH;
			goto exit;
		}
	}

#ifdef CONFIG_LIBUKSP
	uk_stack_chk_guard_setup();
#endif

	print_banner(stdout);
	fflush(stdout);

	/*
	 * Application
	 *
	 * We are calling the application constructors right before calling
	 * the application's main(). All of our Unikraft systems, VFS,
	 * networking stack is initialized at this point. This way we closely
	 * mimic what a regular user application (e.g., BSD, Linux) would expect
	 * from its OS being initialized.
	 */
	uk_pr_info("Pre-init table at %p - %p\n",
		   &__preinit_array_start[0], &__preinit_array_end);
	uk_ctortab_foreach(ctorfn,
			   __preinit_array_start, __preinit_array_end) {
		if (!*ctorfn)
			continue;

		uk_pr_debug("Call pre-init constructor: %p()...\n", *ctorfn);
		(*ctorfn)();
	}

	uk_pr_info("Constructor table at %p - %p\n",
		   &__init_array_start[0], &__init_array_end);
	uk_ctortab_foreach(ctorfn, __init_array_start, __init_array_end) {
		if (!*ctorfn)
			continue;

		uk_pr_debug("Call constructor: %p()...\n", *ctorfn);
		(*ctorfn)();
	}

#if CONFIG_LIBPOSIX_ENVIRON
	envp = environ;
	if (envp) {
		uk_pr_info("Environment variables:\n");
		while (*envp) {
			uk_pr_info("\t%s\n", *envp);
			envp++;
		}
	}
#endif /* CONFIG_LIBPOSIX_ENVIRON */

	uk_pr_info("Calling main(%d, [", argc);
	for (i = 0; i < argc; ++i) {
		uk_pr_info("'%s'", argv[i]);
		if ((i + 1) < argc)
			uk_pr_info(", ");
	}
	uk_pr_info("])\n");
	/* 执行自定义 main 函数。*/
	rc = main(argc, argv);
	uk_pr_info("main returned %d, halting system\n", rc);
	rc = (rc != 0) ? UKPLAT_CRASH : UKPLAT_HALT;

exit:
	ukplat_terminate(rc); /* does not return */
}


```
