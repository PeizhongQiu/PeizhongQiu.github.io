---
title: Unikraft 启动流程
abbrlink: 8e98bdb9
date: 2023-10-08 23:22:09
tags: unikernel 
---

# Unikraft 调试
<!-- more -->
[https://www.youtube.com/watch?v=ANm0diPrWUE](https://www.youtube.com/watch?v=ANm0diPrWUE)
https://users.ece.utexas.edu/~adnan/gdb-refcard.pdf
## KVM
```shell
qemu-system-x86_64  -s -S -kernel build/helloworld_qemu-x86_64 -nographic
gdb --eval-command="target remote :1234" build/helloworld_qemu-x86_64.dbg
```

## linuxu
```shell
gdb build/helloworld_linuxu-x86_64.dbg
```
# Unikraft qemu-x86-64 启动流程
源码版本：0.14.0
## 寻找入口函数
方法 1：阅读 plat/kvm/Linker.uk：
``` linker-script 
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
```shell
readelf -a build/helloworld_qemu-x86_64.dbg
```
查看 Entry point address 选项：
```shell
Entry point address:               0x125f88
```
可知入口地址为 0x125f88。再进行反汇编：
```shell
objdump -ld -C -S build/helloworld_qemu-x86_64.dbg > helloworld.txt
```
查看 helloworld.txt 文件，并找到地址为 0x125f88 的函数：\_multiboot_entry。
## multiboot

https://www.gnu.org/software/grub/manual/multiboot/multiboot.html

## qemu-x86-64 启动源码阅读
由于使用 multiboot 启动，所以启动时的状态为：
- EAX 包含一个魔数；
- EBX 包含 multiboot 信息的 32 位物理地址；
- Flat 4GiB CS 和 DS 段，其中 ES、FS、GS 和 SS 设置为 DS；
- A20 已启用，启用保护模式，禁用分页，禁用中断。

### \_multiboot_entry
从 plat/kvm/x86/multiboot.S 的 \_multiboot_entry 开始阅读。
```asm
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
注意在执行完  \_multiboot_entry 之后，esi 里存储的是包含 multiboot 信息的 32 位物理地址，edi 里存储的是 lcpu_boot_startup_args 的地址，ebx 里存储的是 lcpu_start32 的函数地址。所以函数会跳转到 lcpu_start32。

### lcpu_start32
lcpu_start32 打开 PAE，IA-32e，设置页表，cr0，GDT，将程序跳转到 64 位环境执行。
```asm
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

{% asset_img  页表.jpg 图 1 页表%}

### lcpu_start64

```asm
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
	movq	LCPU_SARGS_ENTRY_OFFSET(%r8), %rax	/* 将函数跳转地址设为 multiboot_entry 函数地址 */
	movq	LCPU_SARGS_STACKP_OFFSET(%r8), %rsp /*将栈地址设为 bss 段地址*/

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

### multiboot_entry
当执行 hello-world 程序时，参数 mi 的值如下所示。mi 各个参数的含义与 multiboot 协议有关，注意在 \_multiboot_entry 定义的 MULTIBOOT_FLAGS 值。
```asm
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
参数 lcpu 值如下所示。其中，idx 表示逻辑 cpu 的顺序索引，id 表示逻辑 cpu 的物理 id。
```shell
(gdb) p/x *lcpu
$4 = {state = 0x1, idx = 0x0, id = 0x0, {s_args = {entry = 0x0, stackp = 0x0}, error_code = 0x0}, arch = {<No data fields>}}
```
lcpu 的状态变化如下图所示。

{% asset_img  lcpu.png 图 2 lcpu%}

下面阅读 multiboot_entry 函数，调试信息为执行 hello-world 程序的信息。
bi->mrds.mrds 的 type 和 flags 取值如下所示。另外，bi->mrds.mrds 按照 pbase 大小有序排列。
```c
/* Memory region types */
#define UKPLAT_MEMRT_ANY		0xffff

#define UKPLAT_MEMRT_FREE		0x0001	/* Uninitialized memory */
#define UKPLAT_MEMRT_RESERVED		0x0002	/* In use by platform */
#define UKPLAT_MEMRT_KERNEL		0x0004	/* Kernel binary segment */
#define UKPLAT_MEMRT_INITRD		0x0008	/* Initramdisk */
#define UKPLAT_MEMRT_CMDLINE		0x0010	/* Command line */
#define UKPLAT_MEMRT_DEVICETREE		0x0020	/* Device tree */
#define UKPLAT_MEMRT_STACK		0x0040	/* Thread stack */

/* Memory region flags */
#define UKPLAT_MEMRF_ALL		0xffff

#define UKPLAT_MEMRF_PERMS		0x0007
#define UKPLAT_MEMRF_READ		0x0001	/* Region is readable */
#define UKPLAT_MEMRF_WRITE		0x0002	/* Region is writable */
#define UKPLAT_MEMRF_EXECUTE		0x0004	/* Region is executable */

#define UKPLAT_MEMRF_UNMAP		0x0010	/* Must be unmapped at boot */
#define UKPLAT_MEMRF_MAP		0x0020	/* Must be mapped at boot */
```

```c
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

### _ukplat_entry
下面阅读 _ukplat_entry 函数，调试信息为执行 hello-world 程序的信息。

```c
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

```c
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

### ukplat_entry_argp 和 ukplat_entry 

```c
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

转到 ukplat_entry 运行。为了方便调试，调试使用 sqlite 应用。

```c
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
	(gdb) p/x &uk_ctortab_start
	$9 = 0x2d9000
	(gdb) p/x &uk_ctortab_end
	$10 = 0x2d9050
	(gdb) p *uk_ctortab_start@10
	$12 = {0x10f9f0 <uk_libparam_params_libposix_environ_ctor>, 0x128ef0 <uk_libparam_params_libukboot_ctor>, 0x12b580 <uk_libparam_params_libuklibparam_ctor>,
		0x141b40 <uk_libparam_params_libvfscore_ctor>, 0x277f00 <_init_ectx_store>, 0x138ab0 <vfscore_init>, 0x278e70 <libvirtio_9p_virtio_register_driver>,
  		0x279ca0 <libvirtio_pci_pci_register_driver>, 0x2788d0 <libukbus_pci_uk_bus_register>, 0x279a00 <libvirtio_bus_uk_bus_register>}
	*/
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
	/* init.h 中定义了相关宏 */
	/*
	(gdb) p/x &uk_inittab_start
	$13 = 0x2d9050
	(gdb) p/x &uk_inittab_end
	$14 = 0x2d9078
	(gdb) p *uk_inittab_start@5
	$15 = {0x131f60 <fdtable_init>, 0x129590 <uk_bus_lib_init>, 0x13e8f0 <pipe_mount_init>, 0x141b80 <vfscore_automount>, 0x110d40 <posix_process_init>}
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
	 /* preinit and init are used in C++ applications. 
	  * Therefore they are empty for sqlite (which is C) */
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
#### uk_ctor

```c
/* 1、ctor 相关宏定义。*/
/* unikraft/plat/common/include/uk/plat/common/common.lds.h */
#define CTORTAB_SECTION							\
	. = ALIGN(__PAGE_SIZE);						\
	uk_ctortab_start = .;						\
	.uk_ctortab :							\
	{								\
		KEEP(*(SORT_BY_NAME(.uk_ctortab[0-9])))			\
	}								\
	uk_ctortab_end = .;
/* include/uk/ctors.h*/
/**
 * Register a Unikraft constructor function that is
 * called during bootstrap (uk_ctortab)
 *
 * @param fn
 *   Constructor function to be called
 * @param prio
 *   Priority level (0 (earliest) to 9 (latest))
 *   Use the UK_PRIO_AFTER() helper macro for computing priority dependencies.
 *   Note: Any other value for level will be ignored
 */
#define __UK_CTORTAB(fn, prio)				\
	static const uk_ctor_func_t			\
	__used __section(".uk_ctortab" #prio)		\
	__uk_ctortab ## prio ## _ ## fn = (fn)

#define _UK_CTORTAB(fn, prio)				\
	__UK_CTORTAB(fn, prio)

#define UK_CTOR_PRIO(fn, prio)				\
	_UK_CTORTAB(fn, prio)

#define UK_CTOR(fn) UK_CTOR_PRIO(fn, UK_PRIO_LATEST)

/* 2、libparam 相关 ctor 宏定义。*/
/* unikraft/lib/uklibparam/include/uk/libparam.h */
/* __LIBNAME__ 是在编译阶段传递的参数。具体可以查看 unikraft/support/build/Makefile.rules 定义的 buildrule_cxx。*/
#define UK_LIBPARAM_PARAMSECTION_NAME       \
	UK_LIBPARAM_CONCAT(UK_LIBPARAM_PARAMSECTION_NAMEPREFIX, __LIBNAME__)
#define UK_LIBPARAM_LIBDESC_CTOR \
	UK_LIBPARAM_CONCAT(UK_LIBPARAM_PARAMSECTION_NAME, _ctor)
static void UK_LIBPARAM_LIBDESC_CTOR(void)
{
	UK_LIBPARAM_LIBDESC.params_len =
		(__sz)((__uptr) &UK_LIBPARAM_PARAMSECTION_ENDSYM -
		       (__uptr) UK_LIBPARAM_PARAMSECTION_STARTSYM) /
		      sizeof(void *);

	if (UK_LIBPARAM_LIBDESC.params_len > 0) {
		UK_LIBPARAM_LIBDESC.params = (struct uk_libparam_param **)
					     UK_LIBPARAM_PARAMSECTION_STARTSYM;
		_uk_libparam_libsec_register(&UK_LIBPARAM_LIBDESC);
	}
}
UK_CTOR_PRIO(UK_LIBPARAM_LIBDESC_CTOR, 0); /* 注册库。*/

/* 3、etcx 和 vsfcore 相关 ctor 定义。*/
/* unikraft/arch/x86/ectx.c */
UK_CTOR_PRIO(_init_ectx_store, 0);

/* unikraft/lib/vfscore/main.c */
UK_CTOR_PRIO(vfscore_init, 1);

/* 4、 PCI 相关 ctor 定义。*/
/* unikraft/plat/common/include/pci/pci_bus.h */
#define PCI_REGISTER_DRIVER(b)                  \
	_PCI_REGISTER_DRIVER(__LIBNAME__, b)
#define PCI_REGISTER_CTOR(ctor)				\
	UK_CTOR_PRIO(ctor, UK_PRIO_AFTER(UK_BUS_REGISTER_PRIO))
#define _PCI_REGISTER_DRIVER(libname, b)				\
	static void						\
	_PCI_REGFNNAME(libname, _pci_register_driver)(void)		\
	{								\
		_pci_register_driver((b));				\
	}								\
	PCI_REGISTER_CTOR(_PCI_REGFNNAME(libname, _pci_register_driver))
/* unikraft/plat/drivers/virtio/virtio_pci.c */
PCI_REGISTER_DRIVER(&virtio_pci_drv);

/* 5、 总线相关 ctor 定义。*/
/* unikraft/lib/ukbus/include/uk/bus.h */
#define UK_BUS_REGISTER(b) \
	_UK_BUS_REGISTER(__LIBNAME__, b, 2)
#define _UK_BUS_REGISTER_CTOR(CTOR, prio)  \
	UK_CTOR_PRIO(CTOR, prio)

#define _UK_BUS_REGISTER(libname, b, prio)			\
	static void						\
	_UK_BUS_REGFNNAME(libname, _uk_bus_register)(void)	\
	{							\
		_uk_bus_register((b));				\
	}							\
	_UK_BUS_REGISTER_CTOR(_UK_BUS_REGFNNAME(libname, _uk_bus_register), prio)

/* unikraft/plat/common/pci_bus.c */
UK_BUS_REGISTER(&ph.b);
/* unikraft/plat/drivers/tap/tap.c */
UK_BUS_REGISTER(&tap_bus);
/* unikraft/plat/drivers/virtio/virtio_bus.c */
UK_BUS_REGISTER(&virtio_bus);

/* unikraft/plat/drivers/include/virtio/virtio_bus.h */
#define VIRTIO_BUS_REGISTER_DRIVER(b)	\
	_VIRTIO_BUS_REGISTER_DRIVER(__LIBNAME__, b)
#define _VIRTIO_REGISTER_CTOR(ctor)	\
	UK_CTOR_PRIO(ctor, UK_PRIO_AFTER(UK_BUS_REGISTER_PRIO))
#define _VIRTIO_BUS_REGISTER_DRIVER(libname, b)				\
	static void							\
	_VIRTIO_BUS_REGFNAME(libname, _virtio_register_driver)(void)	\
	{								\
		_virtio_bus_register_driver((b));			\
	}								\
	_VIRTIO_REGISTER_CTOR(						\
		_VIRTIO_BUS_REGFNAME(					\
		libname, _virtio_register_driver))
/* unikraft/plat/drivers/virtio/virtio_9p.c */
VIRTIO_BUS_REGISTER_DRIVER(&v9p_drv);
/* unikraft/plat/drivers/virtio/virtio_blk.c */
VIRTIO_BUS_REGISTER_DRIVER(&vblk_drv);
/* unikraft/plat/drivers/virtio/virtio_net.c */
VIRTIO_BUS_REGISTER_DRIVER(&vnet_drv);
```

#### uk_inittab
```c
/* 1、uk_inittab 相关定义*/
/* unikraft/include/uk/init.h */
/**
 * Register a Unikraft init function that is
 * called during bootstrap (uk_inittab)
 *
 * @param fn
 *   Initialization function to be called
 * @param class
 *   Initialization class (1 (earliest) to 6 (latest))
 * @param prio
 *   Priority level (0 (earliest) to 9 (latest)), must be a constant.
 *   Use the UK_PRIO_AFTER() helper macro for computing priority dependencies.
 *   Note: Any other value for level will be ignored
 */
#define __UK_INITTAB(fn, base, prio)					\
	static const uk_init_func_t					\
	__used __section(".uk_inittab" #base #prio)			\
	__uk_inittab ## base ## prio ## _ ## fn = (fn)

#define _UK_INITTAB(fn, base, prio)					\
	__UK_INITTAB(fn, base, prio)

#define uk_initcall_class_prio(fn, class, prio)				\
	_UK_INITTAB(fn, class, prio)

/**
 * Define a library initialization. At this point in time some platform
 * component may not be initialized, so it wise to initializes those component
 * to initialized.
 */
#define UK_INIT_CLASS_EARLY 1
#define uk_early_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_EARLY, prio)
/**
 * Define a stage for platform initialization. Platform at this point read
 * all the device and device are initialized.
 */
#define UK_INIT_CLASS_PLAT 2
#define uk_plat_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_PLAT, prio)
/**
 * Define a stage for performing library initialization. This library
 * initialization is performed after the platform is completely initialized.
 */
#define UK_INIT_CLASS_LIB 3
#define uk_lib_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_LIB, prio)
/**
 * Define a stage for filesystem initialization.
 */
#define UK_INIT_CLASS_ROOTFS 4
#define uk_rootfs_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_ROOTFS, prio)
/**
 * Define a stage for device initialization
 */
#define UK_INIT_CLASS_SYS 5
#define uk_sys_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_SYS, prio)
/**
 * Define a stage for application pre-initialization
 */
#define UK_INIT_CLASS_LATE 6
#define uk_late_initcall_prio(fn, prio) \
	uk_initcall_class_prio(fn, UK_INIT_CLASS_LATE, prio)

/* 2、相关实现 */
/* unikraft/lib/posix-socket/driver.c */
#define POSIX_SOCKET_FAMILY_INIT_CLASS UK_INIT_CLASS_EARLY
#define POSIX_SOCKET_FAMILY_INIT_PRIO 0
#define POSIX_SOCKET_FAMILY_REGISTER_PRIO 2

uk_initcall_class_prio(posix_socket_family_lib_init,
		       POSIX_SOCKET_FAMILY_INIT_CLASS,
		       POSIX_SOCKET_FAMILY_INIT_PRIO);

/* unikraft/lib/ukbus/bus.c */
uk_initcall_class_prio(uk_bus_lib_init, UK_BUS_INIT_CLASS, UK_BUS_INIT_PRIO);

/* unikraft/lib/vfscore/fd.c */
uk_early_initcall_prio(fdtable_init, UK_PRIO_EARLIEST);

/* unikraft/lib/uklock/mutex.c */
uk_lib_initcall_prio(mutex_metrics_ctor, 1);

/* unikraft/lib/devfs/devfs_vnops.c */
uk_rootfs_initcall_prio(devfs_automount, 5);

/* unikraft/lib/vfscore/automount.c */
uk_rootfs_initcall_prio(vfscore_automount, 4);

/* unikraft/unikraft/lib/devfs/include/devfs/device.h */
#define devfs_initcall(fn) uk_rootfs_initcall_prio(fn, 3)
/* unikraft/lib/devfs/null.c */
devfs_initcall(devfs_register_null);
/* unikraft/unikraft/lib/devfs/stdout.c*/
devfs_initcall(devfs_register_stdout);
/* unikraft/unikraft/lib/ukswrand/dev.c */
devfs_initcall(devfs_register);
```
#### memory

```c
static struct uk_alloc *heap_init()
{
	struct uk_alloc *a = NULL;
	......
	struct ukplat_memregion_desc *md;
	......
	/* 将 br->mrds 中的内容加入到内存分配器中 */
	ukplat_memregion_foreach(&md, UKPLAT_MEMRT_FREE, 0, 0) {
		uk_pr_debug("Trying %p-%p 0x%02x %s\n",
			    (void *)md->vbase, (void *)(md->vbase + md->len),
			    md->flags,
#if CONFIG_UKPLAT_MEMRNAME
			    md->name
#else /* CONFIG_UKPLAT_MEMRNAME */
			    ""
#endif /* !CONFIG_UKPLAT_MEMRNAME */
			    );

		if (!a)
			a = uk_alloc_init((void *)md->vbase, md->len);
		else
			uk_alloc_addmem(a, (void *)md->vbase, md->len);
	}

	return a;
}
```
首先看一下 uk_alloc 数据结构：
```c
struct uk_alloc {
	/* memory allocation */
	uk_alloc_malloc_func_t malloc;
	uk_alloc_calloc_func_t calloc;
	uk_alloc_realloc_func_t realloc;
	uk_alloc_posix_memalign_func_t posix_memalign;
	uk_alloc_memalign_func_t memalign;
	uk_alloc_free_func_t free;

#if CONFIG_LIBUKALLOC_IFMALLOC
	uk_alloc_free_func_t free_backend;
	uk_alloc_malloc_func_t malloc_backend;
#endif

	/* page allocation interface */
	uk_alloc_palloc_func_t palloc;
	uk_alloc_pfree_func_t pfree;
	/* optional interfaces, but recommended */
	uk_alloc_getsize_func_t maxalloc; /* biggest alloc req. (bytes) */
	uk_alloc_getsize_func_t availmem; /* total memory available (bytes) */
	uk_alloc_getpsize_func_t pmaxalloc; /* biggest alloc req. (pages) */
	uk_alloc_getpsize_func_t pavailmem; /* total pages available */
	/* optional interface */
	uk_alloc_addmem_func_t addmem;

#if CONFIG_LIBUKALLOC_IFSTATS
	struct uk_alloc_stats _stats;
#endif

	/* internal */
	struct uk_alloc *next; /* 形成链表，使每个应用都能根据应用的需要分配内存 */
	__u8 priv[]; /* 内存分配算法的元数据结构从这开始 */
};
```
接下来看一下 uk_alloc_init 实现：
```c
#if CONFIG_LIBUKBOOT_INITBBUDDY
#include <uk/allocbbuddy.h>
#define uk_alloc_init uk_allocbbuddy_init
#elif CONFIG_LIBUKBOOT_INITREGION
#include <uk/allocregion.h>
#define uk_alloc_init uk_allocregion_init
#elif CONFIG_LIBUKBOOT_INITMIMALLOC
#include <uk/mimalloc.h>
#define uk_alloc_init uk_mimalloc_init
#elif CONFIG_LIBUKBOOT_INITTLSF
#include <uk/tlsf.h>
#define uk_alloc_init uk_tlsf_init
#elif CONFIG_LIBUKBOOT_INITTINYALLOC
#include <uk/tinyalloc.h>
#define uk_alloc_init uk_tinyalloc_init
#endif
```
可以看到，uk_alloc_init 的具体实现根据 CONFIG_LIBUKBOOT_INIT* 配置。用户也可以自定义内存分配器。以 uk_allocbbuddy_init 为例：
```c
struct uk_alloc *uk_allocbbuddy_init(void *base, size_t len)
{
	struct uk_alloc *a;
	struct uk_bbpalloc *b; /* uk_bbpalloc 为伙伴算法相关数据结构 */
	size_t metalen;
	uintptr_t min, max;
	unsigned long i;

	min = round_pgup((uintptr_t)base);
	max = round_pgdown((uintptr_t)base + (uintptr_t)len);
	UK_ASSERT(max > min);

	/* Allocate space for allocator descriptor */
	/* 由此分配可知，uk_alloc 和 uk_bbpalloc 相邻 */
	metalen = round_pgup(sizeof(*a) + sizeof(*b));

	/* enough space for allocator available? */
	if (min + metalen > max) {
		uk_pr_err("Not enough space for allocator: %"__PRIsz" B required but only %"__PRIuptr" B usable\n",
			  metalen, (max - min));
		return NULL;
	}
	/* 注意 a 和 b 的地址 */
	a = (struct uk_alloc *)min;
	uk_pr_info("Initialize binary buddy allocator %"__PRIuptr"\n",
		   (uintptr_t)a);
	min += metalen;
	memset(a, 0, metalen);
	b = (struct uk_bbpalloc *)&a->priv;
	
	for (i = 0; i < FREELIST_SIZE; i++) {
		b->free_head[i] = &b->free_tail[i];
		b->free_tail[i].pprev = &b->free_head[i];
		b->free_tail[i].next = NULL;
	}
	b->memr_head = NULL;

	/* initialize and register allocator interface */
	/* 除此之外还有 uk_alloc_init_malloc，用来注册不执行 palloc() 或 pfree() 的内存分配器 */
	uk_alloc_init_palloc(a, bbuddy_palloc, bbuddy_pfree,
			     bbuddy_pmaxalloc, bbuddy_pavailmem,
			     bbuddy_addmem);

	if (max > min) {
		/* add left memory - ignore return value */
		bbuddy_addmem(a, (void *)(min),
				 (size_t)(max - min));
	}

	return a;
}
/* Shortcut for doing a registration of an allocator that only
 * implements palloc(), pfree(), pmaxalloc(), pavailmem(), addmem()
 */
#define uk_alloc_init_palloc(a, palloc_func, pfree_func, pmaxalloc_func, \
			     pavailmem_func, addmem_func)		\
	do {								\
		(a)->malloc         = uk_malloc_ifpages;		\
		(a)->calloc         = uk_calloc_compat;			\
		(a)->realloc        = uk_realloc_ifpages;		\
		(a)->posix_memalign = uk_posix_memalign_ifpages;	\
		(a)->memalign       = uk_memalign_compat;		\
		(a)->free           = uk_free_ifpages;			\
		(a)->palloc         = (palloc_func);			\
		(a)->pfree          = (pfree_func);			\
		(a)->pavailmem      = (pavailmem_func);			\
		(a)->availmem       = (pavailmem_func != NULL)		\
				      ? uk_alloc_availmem_ifpages : NULL; \
		(a)->pmaxalloc      = (pmaxalloc_func);			\
		(a)->maxalloc       = (pmaxalloc_func != NULL)		\
				      ? uk_alloc_maxalloc_ifpages : NULL; \
		(a)->addmem         = (addmem_func);			\
									\
		uk_alloc_stats_reset((a));				\
		uk_alloc_register((a));					\
	} while (0)
```

#### tls
tls 主要由三个部分组成：.tadata，.tbss，TCB。已初始化的线程局部变量在 .tdata 节中分配，未初始化的线程局部变量定义为 COMMON 符号。在 .tbss 节中进行分配。TCB 主要包含线程的相关数据，用户可以根据需要自定义 TCB。但是 TCB 至少要包含一个自指指针 tlsp，放在 TCB 的起始位置。

{% asset_img  tls.png 图 3 tls%}
```C
__sz ukarch_tls_area_size(void)
{
	/* NOTE: X86_64 ABI requires that fs:%0 contains the address of itself,
	 *       to allow certain optimizations. Hence, the overall size of an
	 *       TLS allocation is the aligned up TLS area plus 8 bytes for this
	 *       self-pointer.
	 */
	 /* _tls_start 位于 tls 起始位置，_tls_end 位于 .tbss 末尾 */
	__sz static_tls_len =  ALIGN_UP((__uptr) _tls_end - (__uptr) _tls_start,
					sizeof(void *));
	return static_tls_len + TCB_SIZE;
}
#define TLS_SECTIONS							\
	. = ALIGN(__PAGE_SIZE);						\
	_tls_start = .;							\
	.tdata :							\
	{								\
		*(.tdata)						\
		*(.tdata.*)						\
		*(.gnu.linkonce.td.*)					\
	} UK_SEGMENT_TLS UK_SEGMENT_TLS_LOAD				\
	_etdata = .;							\
	.tbss :								\
	{								\
		*(.tbss)						\
		*(.tbss.*)						\
		*(.gnu.linkonce.tb.*)					\
		*(.tcommon)						\
	}								\
	/*								\
	 * NOTE: Because the .tbss section is zero-sized in the final	\
	 *       ELF image, just setting _tls_end to the end of it	\
	 *       does not give us the the size of the memory area once	\
	 *       loaded, so we use SIZEOF to have it point to the end.	\
	 *       _tls_end is only used to compute the .tbss size.	\
	 */								\
	_tls_end = . + SIZEOF(.tbss);
```
接着，看看 ukarch_tls_area_init 函数。
```c
void ukarch_tls_area_init(void *tls_area)
{
	const __sz tdata_len = (__uptr) _etdata  - (__uptr) _tls_start;
	const __sz tbss_len  = (__uptr) _tls_end - (__uptr) _etdata;
	const __sz padding   = ukarch_tls_area_size() - (tbss_len + tdata_len
							 + TCB_SIZE);

	......

	/* .tdata */
	memcpy((void *)((__uptr) tls_area),
	       _tls_start, tdata_len);

#if CONFIG_LIBCONTEXT_CLEAR_TBSS
	/* clear .tbss and padding */
	memset((void *)((__uptr) tls_area + tdata_len),
	       0x0, tbss_len + padding);
#endif /* CONFIG_LIBCONTEXT_CLEAR_TBSS */

	/* x86_64 ABI requires that fs:%0 contains the address of itself. */
	*((__uptr *) ukarch_tls_tlsp(tls_area))
		= (__uptr) ukarch_tls_tlsp(tls_area);

#if CONFIG_UKARCH_TLS_HAVE_TCB
	ukarch_tls_tcb_init((void *) ukarch_tls_tlsp(tls_area));
#endif /* CONFIG_UKARCH_TLS_HAVE_TCB */

	uk_hexdumpCd(tls_area, ukarch_tls_area_size());
}
```
最后，看看 ukplat_tlsp_set 函数：
```c
/* fs 作用：https://www.kernel.org/doc/html/next/x86/x86_64/fsgs.html */
#define set_tls_pointer(ptr) wrmsrl(X86_MSR_FS_BASE, ptr)
void ukplat_tlsp_set(__uptr tlsp)
{
	set_tls_pointer(tlsp);
}
```

#### sched
首先看看 uk_sched 数据结构内容：
```c
struct uk_sched {
	uk_sched_yield_func_t yield;

	uk_sched_thread_add_func_t      thread_add;
	uk_sched_thread_remove_func_t   thread_remove;
	uk_sched_thread_blocked_func_t  thread_blocked;
	uk_sched_thread_woken_func_t    thread_woken;
	uk_sched_idle_thread_func_t     idle_thread;

	uk_sched_start_t sched_start;

	/* internal */
	bool is_started;
	struct uk_thread_list thread_list;
	struct uk_thread_list exited_threads;
	struct uk_alloc *a;       /**< default allocator for struct uk_thread */
	struct uk_alloc *a_stack; /**< default allocator for stacks */
	struct uk_alloc *a_uktls; /**< default allocator for TLS+ectx */
	struct uk_sched *next;
};

struct schedcoop {
	struct uk_sched sched;
	struct uk_thread_list run_queue;
	struct uk_thread_list sleep_queue;

	struct uk_thread idle;
	__nsec idle_return_time;
	__nsec ts_prev_switch;
};
```

```c
struct uk_sched *uk_schedcoop_create(struct uk_alloc *a)
{
	struct schedcoop *c = NULL;
	int rc;

	uk_pr_info("Initializing cooperative scheduler\n");
	c = uk_zalloc(a, sizeof(struct schedcoop));
	if (!c)
		goto err_out;

	UK_TAILQ_INIT(&c->run_queue);
	UK_TAILQ_INIT(&c->sleep_queue);

	/* Create idle thread */
	rc = uk_thread_init_fn1(&c->idle,
				idle_thread_fn, (void *) c,
				a, STACK_SIZE,
				a, false,
				NULL,
				"idle",
				NULL,
				NULL);
	if (rc < 0)
		goto err_free_c;

	c->idle.sched = &c->sched;

	uk_sched_init(&c->sched,
			schedcoop_start,
			schedcoop_schedule,
			schedcoop_thread_add,
			schedcoop_thread_remove,
			schedcoop_thread_blocked,
			schedcoop_thread_woken,
			schedcoop_idle_thread,
			a);

	/* Add idle thread to the scheduler's thread list */
	UK_TAILQ_INSERT_TAIL(&c->sched.thread_list, &c->idle, thread_list);

	return &c->sched;

err_free_c:
	uk_free(a, c);
err_out:
	return NULL;
}

int uk_sched_start(struct uk_sched *s)
{
	struct uk_thread *main_thread;
	uintptr_t tlsp;
	int ret;

	UK_ASSERT(s);
	UK_ASSERT(s->sched_start);
	UK_ASSERT(!s->is_started);
	UK_ASSERT(!uk_thread_current()); /* No other thread runs */

	/* Allocate an `uk_thread` instance for current context
	 * NOTE: We assume that if we have a TLS pointer, it points to
	 *       an TLS that is derived from the Unikraft TLS template.
	 */
	tlsp = ukplat_tlsp_get();
	main_thread = uk_thread_create_bare(s->a,
					    0x0, 0x0, tlsp, !(!tlsp), false,
					    "init", NULL, NULL);
	if (!main_thread) {
		ret = -ENOMEM;
		goto err_out;
	}
	main_thread->sched = s;

	/* Because `main_thread` acts as container for storing the current
	 * context, it does not have IP and SP set. We have to manually mark
	 * the thread as RUNNABLE.
	 */
	uk_thread_set_runnable(main_thread);

	/* Set main_thread as current scheduled thread */
	ukplat_per_lcpu_current(__uk_sched_thread_current) = main_thread;

	/* Add main to the scheduler's thread list */
	UK_TAILQ_INSERT_TAIL(&s->thread_list, main_thread, thread_list);

	/* Enable scheduler, like time slicing, etc. and notify that `s`
	 * has an (already) scheduled thread
	 */
	ret = s->sched_start(s, main_thread);
	if (ret < 0)
		goto err_unset_thread_current;
	s->is_started = true;
	return 0;

err_unset_thread_current:
	ukplat_per_lcpu_current(__uk_sched_thread_current) = NULL;
	uk_thread_release(main_thread);
err_out:
	return ret;
}
```
# Unikraft linuxu 64 位启动流程
## 寻找入口函数
方法 1：阅读 unikraft/plat/linuxu/Linker.uk：
```linker-script
LINUXU_LDFLAGS-y += -Wl,-e,_liblinuxuplat_start
```

从这里可以看出，当入口函数为 \_liblinuxuplat_start。

方法 2：阅读 elf 文件。
以 helloworld 为例，使用以下命令：
```shell
readelf -a build/helloworld_linuxu-x86_64.dbg
```
查看 Entry point address 选项：
```shell
Entry point address:               0x401090
```
可知入口地址为 0x401090。再进行反汇编：
```shell
objdump -ld -C -S build/helloworld_linuxu-x86_64.dbg > helloworld.txt
```
查看 helloworld.txt 文件，并找到地址为 0x401090 的函数：\_liblinuxuplat_start。

## _liblinuxuplat_start
```asm
.global _liblinuxuplat_start
_liblinuxuplat_start:
	xorl %ebp, %ebp		# mark the outmost frame (clear the frame pointer)

	popq %rdi		# argc as first argument
	movq %rsp, %rsi		# move argv to rsi, the second parameter in x86_64 abi

	# ignore environ for now

	andq $~15, %rsp		# align stack to 16-byte boundary

	# Run _liblinuxuplat_entry(argc, argv)
	callq *_liblinuxuplat_entry@GOTPCREL(%rip)

	# Protection
_liblinuxuplat_start_err:
	jmp *_liblinuxuplat_start_err(%rip)
```
转到 _liblinuxuplat_entry 执行。
## _liblinuxuplat_entry
由于 linuxu 是在 linux 用户态执行，所以不用像 qemu 启动那样复杂，不用设置启动协议，设置页表等功能，可以直接使用原来内核的相关功能。
```c
void _liblinuxuplat_entry(int argc, char *argv[])
{
	/*
	 * Initialize platform console
	 */
	_liblinuxuplat_init_console();

	__linuxu_plat_heap_init();

	__linuxu_plat_initrd_init();

	/*
	 * Enter Unikraft
	 */
	ukplat_entry(argc, argv);
}
```
ukplat_entry 过程与 上面一致。


简单分析 \_\_linuxu_plat_heap_init 和 \_\_linuxu_plat_initrd_init。
```c
#define MB2B		(1024 * 1024)
static __u32 heap_size = CONFIG_LINUXU_DEFAULT_HEAPMB;
static void __linuxu_plat_heap_init(void)
{
	void *pret;
	__u32 len;
	int rc;

	len = heap_size * MB2B;
	uk_pr_info("Allocate memory for heap (%u MiB)\n", heap_size);

	/**
	 * Allocate heap memory
	 */
	if (len > 0) {
		pret = sys_mapmem(NULL, len);
		if (PTRISERR(pret)) {
			rc = PTR2ERR(pret);
			uk_pr_err("Failed to allocate memory for heap: %d\n",
				  rc);
		} else {
			/* 将新分配的内存加入到 bi->mrds */
			rc = ukplat_memregion_list_insert(
				&ukplat_bootinfo_get()->mrds,
				&(struct ukplat_memregion_desc){
					.vbase = (__vaddr_t)pret,
					.pbase = (__paddr_t)pret,
					.len   = len,
					.type  = UKPLAT_MEMRT_FREE,
					.flags = UKPLAT_MEMRF_READ |
						 UKPLAT_MEMRF_WRITE |
						 UKPLAT_MEMRF_MAP,
				});
			if (unlikely(rc < 0))
				uk_pr_err("Failed to add heap memory region descriptor.");
		}
	}
}

static void __linuxu_plat_initrd_init(void)
{
	void *pret;
	__u32 len;
	int rc;
	struct k_stat file_info;

	if (!initrd) {
		uk_pr_debug("No initrd present.\n");
	} else {
		uk_pr_debug("Mapping in initrd file: %s\n", initrd);
		int initrd_fd = sys_open(initrd, K_O_RDONLY, 0);

		if (initrd_fd < 0)
			uk_pr_crit("Failed to open %s for initrd\n", initrd);

		/**
		 * Find initrd file size
		 */
		if (sys_fstat(initrd_fd, &file_info) < 0) {
			uk_pr_crit("sys_fstat failed for initrd file\n");
			sys_close(initrd_fd);
		}
		len = file_info.st_size;
		/**
		 * Allocate initrd memory
		 */
		if (len > 0) {
			pret = sys_mmap(NULL, len,
					PROT_READ | PROT_WRITE | PROT_EXEC,
					MAP_PRIVATE, initrd_fd, 0);
			if (PTRISERR(pret)) {
				rc = PTR2ERR(pret);
				uk_pr_crit("Failed to memory-map initrd: %d\n",
					   rc);
				sys_close(initrd_fd);
			}
			/* 将为 initrd 分配的内存加入到 bi->mrds */
			rc = ukplat_memregion_list_insert(
				&ukplat_bootinfo_get()->mrds,
				&(struct ukplat_memregion_desc){
					.vbase = (__vaddr_t)pret,
					.pbase = (__paddr_t)pret,
					.len   = len,
					.type  = UKPLAT_MEMRT_INITRD,
					.flags = UKPLAT_MEMRF_READ |
						 UKPLAT_MEMRF_WRITE |
						 UKPLAT_MEMRF_MAP,
				});
			if (unlikely(rc < 0))
				uk_pr_err("Failed to add initrd memory region descriptor.");
		} else {
			uk_pr_info("Ignoring empty initrd file.\n");
			sys_close(initrd_fd);
		}
	}
}
```