---
title: Unikraft 安装与使用
tags: unikernel
abbrlink: 39b93952
date: 2023-10-23 23:45:57
---

# Unikraft 安装
<!-- more -->
源代码地址：
https://github.com/unikraft/unikraft

安装 kraft： 
```shell
curl --proto '=https' --tlsv1.2 -sSf https://get.kraftkit.sh | sh
```

参考网页：
[https://github.com/unikraft/kraftkit](https://github.com/unikraft/kraftkit)
https://unikraft.org/docs/getting-started

# Unikraft 使用
接下来以 sqlite qemu-x86-64 为例说明 unikraft 的使用。
https://github.com/unikraft/app-sqlite
https://github.com/unikraft/app-sqlite/releases/tag/RELEASE-0.15.0

## 快速使用指南
方法一：kraft
```Shell
git clone https://github.com/unikraft/app-sqlite sqlite
cd sqlite/
kraft build # 或者直接使用 kraft build --target sqlite-qemu-x86_64-initrd -j $(nproc)
kraft run --initrd rootfs/
```

方法二：make
```Shell
git clone https://github.com/unikraft/app-sqlite sqlite
cd sqlite/
./scripts/setup.sh
wget https://raw.githubusercontent.com/unikraft/app-testing/staging/scripts/generate.py -O scripts/generate.py
chmod a+x scripts/generate.py # 根据 defconfig 目录下的文件，生成构建和运行脚本。
./scripts/generate.py
./scripts/build/make-qemu-x86_64-9pfs.sh 
./scripts/run/qemu-x86_64-9pfs.sh 
```

## app-sqlite 目录
首先在编译链接之前，看看 sqlite 目录结构：

``` shell
$ tree sqlite/
sqlite/
├── defconfigs # defconfig 里面存放的是启动该实例需要的最小配置文件，如果需要额外的配置，可以使用 make menuconfig 来配置
│   ├── fc-x86_64-initrd
│   ├── qemu-arm64-9pfs
│   ├── qemu-arm64-initrd
│   ├── qemu-x86_64-9pfs
│   └── qemu-x86_64-initrd
├── kraft.cloud.yaml
├── kraft.yaml # 该文件用于使用 kraft 配置、构建应用程序并将其打包为 Unikraft unikernel
├── Makefile # 用于指定主 Unikraft 存储库相对于应用程序存储库的位置，以及应用程序需要使用的任何外部库的存储库
├── Makefile.uk # 用于指定要构建的源（以及可选地从何处获取它们），包括路径、标志和任何特定于应用程序的目标
├── README.md
├── rootfs # rootfs 是 qemu 启动虚拟机时要传递的参数，在这个例子中，该目录下存放的是数据库数据
│   ├── chinook.db
│   └── script.sql
└── scripts # 准备和运行脚本
    ├── run.yaml
    └── setup.sh # 下载需要的库
```

## 方法一
### Kraftfile
参考链接：https://unikraft.org/docs/cli/reference/kraftfile/v0.5
Kraftfile 可以有以下命名：Kraftfile，kraft.yaml，kraft.yml。
简单看一下 kraft.yaml 内容：
```shell
specification: '0.5' # 所有 Kraftfile 必须包含一个顶级 specification 属性，kraft 使用该属性来验证并正确解析文件的其余部分。最新的规范号是v0.5。
name: sqlite # 可以指定应用程序名称。当未指定 name 属性时，将使用目录的基本名称作为名称
unikraft: # 必须指定 unikraft 属性，并用于定义 Unikraft core 的源代码位置
  version: stable # 可以是 stable 也可以是 unikraft 的某一次 Git 提交：“v0.14.0”，“70bc0af”
  # source 指定源代码位置，source 和 version 参数一起确定源代码。比如：
  # source: https://github.com/unikraft/unikraft.git
  # source: path/to/unikraft
  kconfig:
    - CONFIG_LIBUKLIBPARAM=y
    - CONFIG_LIBUKMMAP=y
    - CONFIG_LIBPOSIX_SYSINFO=y
targets: # 目标被定义为生成的 unikernel 所指向的特定目的地，并且至少由特定平台（例如 qemu 或 firecracker）和架构（例如 x86_64 或 arm64）元组组成。 根据用例，一个项目可以有多个目标，但必须至少有一个。
  - name: sqlite-qemu-x86_64-9pfs
    architecture: x86_64
    platform: qemu
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_9PFS=y
      - CONFIG_LIBVFSCORE_ROOTFS="9pfs"
      - CONFIG_LIBVFSCORE_ROOTDEV="fs1"
  - name: sqlite-qemu-x86_64-initrd
    architecture: x86_64
    platform: qemu
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_INITRD=y
      - CONFIG_LIBVFSCORE_ROOTFS="initrd"
  - name: sqlite-qemu-arm64-9pfs
    architecture: arm64
    platform: qemu
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_ARCH_ARM_64=y
      - CONFIG_MCPU_ARM64_NONE=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_9PFS=y
      - CONFIG_LIBVFSCORE_ROOTFS="9pfs"
      - CONFIG_LIBVFSCORE_ROOTDEV="fs1"
  - name: sqlite-qemu-arm64-initrd
    architecture: arm64
    platform: qemu
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_ARCH_ARM_64=y
      - CONFIG_MCPU_ARM64_NONE=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_INITRD=y
      - CONFIG_LIBVFSCORE_ROOTFS="initrd"
  - name: sqlite-fc-x86_64-initrd
    architecture: x86_64
    platform: fc
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_KVM_BOOT_PROTO_LXBOOT=y
      - CONFIG_KVM_VMM_FIRECRACKER=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_INITRD=y
      - CONFIG_LIBVFSCORE_ROOTFS="initrd"
  - name: sqlite-fc-arm64-initrd
    architecture: arm64
    platform: fc
    kconfig:
      - CONFIG_PLAT_KVM=y
      - CONFIG_KVM_BOOT_PROTO_LXBOOT=y
      - CONFIG_KVM_VMM_FIRECRACKER=y
      - CONFIG_ARCH_ARM_64=y
      - CONFIG_MCPU_ARM64_NONE=y
      - CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
      - CONFIG_LIBVFSCORE_ROOTFS_INITRD=y
      - CONFIG_LIBVFSCORE_ROOTFS="initrd"
libraries: # 其他第三方库可以指定为构建的一部分，并以 map-format 列出。与 unikraft 属性类似，每个库都可以指定源、版本和一组 kconfig 选项
  musl:
    version: stable
  sqlite:
    version: stable
    kconfig:
      - CONFIG_LIBSQLITE_MAIN_FUNCTION=y
```


### kraft
Kraftfile 是静态配置文件，用于使用 kraft 以编程方式构建和打包 unikernel。 该文件包含有关 Unikraft 核心构建系统、第三方库、用于构建的所有配置选项以及应用程序的可能目标列表的信息。 对于所有组件，可以定义 KConfig 选项来设置选项及其各自的值。要发现更多选项或以图形方式设置内容，可以调用：
``` shell
kraft menu
```
构建 unikernel 使用以下命令：
```shell
kraft build [FLAGS] [SUBCOMMAND|DIR]

$ kraft build --help
Build a Unikraft unikernel.

The default behaviour of `kraft build` is to build a project.  Given no
arguments, you will be guided through interactive mode.


USAGE
  kraft build [FLAGS] [SUBCOMMAND|DIR]

FLAGS
      --all                      Build all targets
  -m, --arch string              Filter the creation of the build by architecture of known targets
      --build-log string         Use the specified file to save the output from the build
  -c, --config string            Override the path to the KConfig .config file
      --config-dir string        Path to KraftKit config directory
      --containerd-addr string   Address of containerd daemon socket
      --dbg                      Build the debuggable (symbolic) kernel image instead of the stripped image
      --editor string            Set the text editor to open when prompt to edit a file
      --events-pid-file string   Events process ID used when running multiple unikernels
      --git-protocol string      Preferred Git protocol to use (default "https")
  -h, --help                     help for build
      --http-unix-sock string    When making HTTP(S) connections, pipe requests via this shared socket
  -j, --jobs int                 Allow N jobs at once
  -K, --kraftfile string         Set an alternative path of the Kraftfile
      --log-level string         Log level verbosity (default "info")
      --log-timestamps           Enable log timestamps
      --log-type string          Log type (default "fancy")
      --manifests-dir string     Path to Unikraft manifest cache
  -F, --no-cache                 Force a rebuild even if existing intermediate artifacts already exist
      --no-check-updates         Do not check for updates
      --no-configure             Do not run Unikraft's configure step before building
      --no-emojis                Do not use emojis in any console output
      --no-fast                  Do not use maximum parallelization when performing the build
      --no-fetch                 Do not run Unikraft's fetch step before building
      --no-parallel              Do not run internal tasks in parallel
      --no-prompt                Do not prompt for user interaction
      --no-pull                  Do not pull packages before invoking Unikraft's build system
      --no-update                Do not update package index before running the build
      --pager string             System pager to pipe output to
  -p, --plat string              Filter the creation of the build by platform of known targets
      --plugins-dir string       Path to KraftKit plugin directory
      --runtime-dir string       Directory for placing runtime files (e.g. pidfiles)
      --sources-dir string       Path to Unikraft component cache
  -t, --target string            Build a particular known target
      --with-manifest strings    Paths to package or component manifests (default [https://manifests.kraftkit.sh/index.yaml])
      --with-mirror strings      Paths to mirrors of Unikraft component artifacts

EXAMPLES
  # Build the current project (cwd)
  $ kraft build

  # Build path to a Unikraft project
  $ kraft build path/to/app
```
运行应用命令：
```shell
kraft run [FLAGS] PROJECT|PACKAGE|BINARY -- [APP ARGS]
# Run a built target in the current working directory project:
$ kraft run

# Run a specific target from a multi-target project at the provided project directory:
$ kraft run -t TARGET path/to/project

# Run a specific kernel binary:
$ kraft run --arch x86_64 --plat qemu path/to/kernel-x86_64-qemu

# Run an OCI-compatible unikernel, mapping port 8080 on the host to port 80 in the unikernel:
$ kraft run -p 8080:80 unikraft.org/nginx:latest

# Attach the unikernel to an existing network kraft0 backed by the bridge driver:
$ kraft run --network bridge:kraft0

# Run a Linux userspace binary in POSIX-/binary-compatibility mode:
$ kraft run a.out

# Supply an initramfs CPIO archive file to the unikernel for its rootfs:
$ kraft run --initrd ./initramfs.cpio

# Supply a path which is dynamically serialized into an initramfs CPIO archive:
$ kraft run --initrd ./path/to/rootfs

# Mount a bi-directional path from on the host to the unikernel mapped to /dir:
$ kraft run -v ./path/to/dir:/dir

# Supply a read-only root file system at / via initramfs CPIO archive and mount a bi-directional volume at /dir:
$ kraft run --initrd ./initramfs.cpio --volume ./path/to/dir:/dir

# Customize the default content directory of the official Unikraft NGINX OCI-compatible unikernel and map port 8080 to localhost:
$ kraft run -v ./path/to/html:/nginx/html -p 8080:80 unikraft.org/nginx:latest

# 参数设置
-m, --arch string            Set the architecture
      --as string              Force a specific runner
  -d, --detach                 Run unikernel in background
  -W, --disable-acceleration   Disable acceleration of CPU (usually enables TCG)
  -h, --help                   help for run
      --initrd string          Use the specified initrd (readonly)
  -i, --interactive            Keep stdin open even if not attached
      --ip string              Assign the provided IP address
  -a, --kernel-arg strings     Set additional kernel arguments
      --kraftfile string       Set an alternative path of the Kraftfile
      --mac string             Assign the provided MAC address
  -M, --memory string          Assign MB memory to the unikernel (default "64M")
  -n, --name string            Name of the instance
      --network string         Attach instance to the provided network in the format <driver>:<network>, e.g. bridge:kraft0
      --no-start               Do not start the machine
      --plat string            Set the platform virtual machine monitor driver (default "auto")
  -p, --port stringArray       Publish a machine's port(s) to the host
      --rm                     Automatically remove the unikernel when it shutsdown
      --rootfs string          Specify a path to use as root file system (can be volume or initramfs)
      --symbolic               Use the debuggable (symbolic) unikernel
  -t, --target string          Explicitly use the defined project target
      --tty                    Allocate a pseudo-TTY
  -v, --volume strings         Bind a volume to the instance
```

## 方法二
首先看一下 setup.sh 的内容：
```shell
#!/bin/bash

test ! -d "workdir" && echo "Cloning repositories..." || true
test ! -d "workdir/unikraft" && git clone https://github.com/unikraft/unikraft workdir/unikraft || true
test ! -d "workdir/libs/sqlite" && git clone https://github.com/unikraft/lib-sqlite workdir/libs/sqlite || true
test ! -d "workdir/libs/musl" && git clone https://github.com/unikraft/lib-musl workdir/libs/musl || true
```
setup.sh 内部下载了三个仓库：unikraft，lib-musl，lib-sqlite。以 lib-sqlite 为例，看一下目录结构：
```shell
workdir/libs$ tree sqlite
sqlite
├── CODING_STYLE.md
├── Config.uk
├── CONTRIBUTING.md
├── COPYING.md
├── include
│   └── sqlite_cfg.h
├── main.c # glue code
├── MAINTAINERS.md
├── Makefile.uk
└── README.md
```

参考链接：https://unikraft.org/docs/internals/build-system
### Makefile
**在方法二中，我们将重点关注属于应用程序（即不属于库）的文件。 移植库的过程非常相似，只需将变量名的 APP 前缀更改为 LIB 即可。**
Makefile 通常简短且简单。 对于大多数应用程序，Makefile 不应超过以下内容：
```makefile
UK_ROOT  ?= $(PWD)/workdir/unikraft
UK_LIBS  ?= $(PWD)/workdir/libs
UK_PLATS ?= $(PWD)/workdir/plats
LIBS  := $(UK_LIBS)/lib1:$(UK_LIBS)/lib2:$(UK_LIBS)/libN
PLATS ?=

all:
        @make -C $(UK_ROOT) A=$(PWD) L=$(LIBS) P=$(PLATS)

$(MAKECMDGOALS):
        @make -C $(UK_ROOT) A=$(PWD) L=$(LIBS) P=$(PLATS) $(MAKECMDGOALS)
```
这个 makefile 等价于以下命令：
```make
make -C workdir/unikraft A=. L=workdir/libs/lib1:workdir/libs/lib2:workdir/libs/libN
```

### Makefile.uk

Unikraft 构建系统提供了许多函数和宏，使编写 Makefile.uk 文件变得更加容易。另外，请确保定义的所有变量均以 APP[NAME]_ 或 LIB[NAME]_（例如 APPHELLOWORLD_）开头，以便它们正确命名。

首先要做的是调用 Unikraft addlib 函数向系统注册应用程序，所有内容在 UNikraft 中最终都是一个库。此函数调用还将包含应用程序源的目录路径填入 APPNAME_BASE，将包含应用程序构建目录的目录路径填入 APPNAME_BUILD：
```makefile
$(eval $(call addlib,appname))
```
name 被替换成应用名称。

如果主应用程序代码需要从远程服务器下载，下一步是设置一个变量以指向包含应用程序源（或对象或预链接库）的 URL，并调用 Unikraft fetch 函数来下载并解压这些源（提取文件的目录路径将被填入 APPNAME_ORIGIN）：
```makefile
APPNAME_ZIPNAME = myapp-v0.0.1
APPNAME_URL = http://app.org/$(APPNAME_ZIPNAME).zip
$(eval $(call fetch,appname,$(APPNAME_URL)))
```
接下来设置对 Unikraft patch 函数的调用。 即使还没有任何补丁，也最好进行此设置以备以后需要。
```makefile
APPNAME_PATCHDIR=$(APPNAME_BASE)/patches
$(eval $(call patch,appname,$(APPNAME_PATCHDIR),$(APPNAME_ZIPNAME)))
```
完成所有这些后，就可以开始测试了：
```makefile
$ make menuconfig
$ make
```
应该看到 Unikraft 下载应用程序并将其解压。对通过菜单或 makefile 指定的库会执行相同的操作。但是，目前还没有为应用程序构建任何内容，因为尚未指定任何要构建的源文件。构建时，Unikraft 会创建一个 build/ 目录，并将所有临时文件和目标文件放在该目录；应用程序源文件位于 build/origin/[tarballname]/ 下。

为了告诉 Unikraft 要构建哪些源文件，要将文件添加到 APPNAME_SRCS-y 变量中：
```makefile
APPNAME_SRCS-y += $(APPNAME_BASE)/path_to_src/myfile.c
```
对于源文件，Unikraft 到目前为止支持 C (.c)、C++（.cc、.cxx、.cpp）和汇编文件（.s、.S）。

如果有预编译的目标文件，可以添加它们（但由于预编译目标文件的编译标志可能不兼容，应该小心处理）：
```makefile
APPNAME_OBJS-y += $(APPNAME_BASE)/path_to_src/myobj.o
```
还可以使用 APPNAME_OBJS-y 添加预构建的库（如 .o 或 .a）：
```makefile
APPNAME_OBJS-y += $(APPNAME_BASE)/path_to_lib/mylib.a
```
一旦指定了所有源文件（以及可选的二进制文件），通常还需要指定头文件路径和编译标志：
```makefile
# Include paths
APPNAME_ASINCLUDES  += -I$(APPNAME_BASE)/path_to_include/include [for assembly files]
APPNAME_CINCLUDES   += -I$(APPNAME_BASE)/path_to_include/include [for C files]
APPNAME_CXXINCLUDES += -I$(APPNAME_BASE)/path_to_include/include [for C++ files]

# Flags for application sources
APPNAME_ASFLAGS-y   += -DFLAG_NAME1 ... -DFLAG_NAMEN [for assembly files]
APPNAME_CFLAGS-y    += -DFLAG_NAME1 ... -DFLAG_NAMEN [for C files]
APPNAME_CXXFLAGS-y  += -DFLAG_NAME1 ... -DFLAG_NAMEN [for C++ files]
```
完成所有这些后，可以保存 Makefile.uk 并输入 make。假设所选的 Unikraft 库提供了应用程序所需的所有支持，Unikraft 应该将所有内容编译并链接在一起，并为菜单中指定的每个目标平台输出一个镜像。

除了提到的所有功能之外，应用程序可能需要在下载并解压源代码之后、编译之前执行许多其他任务（例如，运行配置脚本或从源文件生成源代码的自定义脚本）。为了支持这一点，Unikraft 提供了 UK_PREPARE 变量，连接到内部 prepare 目标，可以将其设置为临时标记文件，并从那里设置为 Makefile.uk 文件中的目标。 例如：
```makefile
$(APPNAME_BUILD)/.prepared: [dependencies to further targets]
       cmd && $(TOUCH) $@

UK_PREPARE += $(APPNAME_BUILD)/.prepared
```
通过这种方式，可以确保在进行任何编译之前运行 cmd。 如果使用 fetch，请添加 \$(APPNAME_BUILD)/.origin 作为依赖项。 如果使用 patch，请添加 \$(APPNAME_BUILD)/.patched。

此外，可能会发现有必要为特定源文件指定编译标志或头文件。 Unikraft 通过以下语法支持这一点：
```makefile
APPNAME_SRCS-y += $(APPNAME_BASE)/filename.c
APPNAME_FILENAME_FLAGS-y += -DFLAG
APPNAME_FILENAME_INCLUDES-y += -Iextra/include
```
还可以使用不同的标志多次编译单个源文件。 对于这种情况，Unikraft 支持变体：
```makefile
APPNAME_SRCS-y += $(APPNAME_BASE)/filename.c|variantname
APPNAME_FILENAME_VARIANTNAME_FLAGS-y += -DFLAG2
APPNAME_FILENAME_VARIANTNAME_INCLUDES-y += -Iextra/include
```
最后，可能还需要提供 “glue” 代码，例如 Unikraft 需要实现一个 main() 函数来调用应用程序的 main 或 init 例程。建议将任何此类文件放在应用程序的主目录 (APPNAME_BASE) 中，并将它们可能依赖的头文件放在 APPNAME_BASE/include/ 下。 当然，不要忘记添加源文件并包含 Makefile.uk 的路径。
到目前为止，名称范围内的保留变量名称:
```makefile
APPNAME_BASE                              - Path to source base
APPNAME_BUILD                             - Path to target build dir
APPNAME_EXPORTS                           - Path to the list of exported symbols
                                            (default is '$(APPNAME_BASE)/exportsyms.uk')
APPNAME_ORIGIN                            - Path to extracted archive
                                            (when fetch or unarchive was used)
APPNAME_CLEAN APPNAME_CLEAN-y             - List of files to clean additional
                                            on make clean
APPNAME_SRCS APPNAME_SRCS-y               - List of source files to be
                                            compiled
APPNAME_OBJS APPNAME_OBJS-y               - List of object files to be linked
                                            for the library
APPNAME_OBJCFLAGS APPNAME_OBJCFLAGS-y     - link flags (e.g., define symbols
                                            as internal)
APPNAME_CFLAGS APPNAME_CFLAGS-y           - Flags for C files of the library
APPNAME_CXXFLAGS APPNAME_CXXFLAGS-y       - Flags for C++ files of the library
APPNAME_ASFLAGS APPNAME_ASFLAGS-y         - Flags for assembly files of the
                                            library
APPNAME_CINCLUDES APPNAME_CINCLUDES-y     - Includes for C files of the
                                            library
APPNAME_CXXINCLUDES APPNAME_CXXINCLUDES-y - Includes for C++ files of the
                                            library
APPNAME_ASINCLUDES APPNAME_ASINCLUDES-y   - Includes for assembly files of
                                            the library
APPNAME_FILENAME_FLAGS                    - Flags for a *specific* source file
APPNAME_FILENAME_FLAGS-y                    of the library (not exposed to its
                                            variants)
APPNAME_FILENAME_INCLUDES                 - Includes for a *specific* source
APPNAME_FILENAME_INCLUDES-y                 file of the library (not exposed
                                            to its variants)
APPNAME_FILENAME_VARIANT_FLAGS            - Flags for a *specific* source file
APPNAME_FILENAME_VARIANT_FLAGS-y            and variant of the library
APPNAME_FILENAME_VARIANT_INCLUDES         - Includes for a *specific* source
APPNAME_FILENAME_VARIANT_INCLUDES-y         file and variant of the library
```
### Config.uk
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/Documentation/kbuild/kconfig-language.rst

https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html

Unikraft 的配置系统是基于 Linux 的 KConfig 系统。使用 Config.uk，可以为应用程序定义可能的配置并定义对其他库的依赖关系。Unikraft 包含此文件来呈现基于菜单的配置，以及加载和存储配置（.config）。该文件中定义的每个设置都将全局填充为所有 Makefile.uk 文件的设置变量，以及使用 “#include <uk/config.h>” 时源代码中定义的宏。请确保所有设置都正确命名。它们应该以 [APPNAME]_ 开头（例如 APPHELLOWORLD_）。另请注意，一些变量名称是为每个应用程序或库命名空间预定义的（请参阅 Makefile.uk）。

作为最佳实践，建议在文件开始时定义设置为 y 的在 menuconfig 屏幕中不可见的布尔类型依赖项：
```makefile
### Invisible option for dependencies
config APPHELLOWORLD_DEPENDENCIES
    bool
    default y
    # dependencies with `select`:
    select LIBNOLIBC if !HAVE_LIBC
    select LIB1
    select LIB2
```
应用程序的任何其他设置都可以是布尔值、整数或字符串类型。 甚至可以定义下拉选择。请注意，Unikraft 没有 tristates（对于内核模块，m 选项不可用）。
### exportsyms.uk
Unikraft 为每个库提供单独的命名空间。这意味着每个函数和变量仅在内部可见和可链接。

要使符号对其他库可见，请将其添加到此 exportsyms.uk 文件中。每行一个符号名称。行注释可以通过 **#** 引入。

如果您正在编写应用程序，则需要将程序入口点添加到此文件中。很可能那里应该没有其他东西。对于库，必须列出所有外部 API 函数。

为了文件结构的一致性，不建议更改此符号文件的默认路径。可以通过定义 APPNAME_EXPORTS 变量来覆盖它。该路径必须是绝对路径（可以使用 \$(APPNAME_BASE) 引用应用程序源的基本目录）或相对于 Unikraft 源目录。

如果给定库中不存在 exportsyms.uk 文件，则默认情况下将导出所有符号。

### extra.ld
如果库/应用程序需要最终可执行映像中的一个 section，请编辑 Makefile.uk 以添加:
```makefile
LIBYOURAPPNAME_SRCS-$(CONFIG_LIBYOURAPPNAME) += $(LIBYOURAPPNAME_BASE)/extra.ld
```
extra.ld 的例子：
```asm
SECTIONS
{
    .uk_fs_list : {
         PROVIDE(uk_fslist_start = .);
         KEEP (*(.uk_fs_list))
         PROVIDE(uk_fslist_end = .);
    }
}
INSERT AFTER .text;
```
这会在 .text section 后添加 .uk_fs_list section。

### make 和 run
以 script/build/make-qemu-x86_64-initrd.sh 和 script/run/qemu-x86_64-initrd.sh
```shell
#!/bin/sh

make distclean
UK_DEFCONFIG=$(pwd)/defconfigs/qemu-x86_64-initrd make defconfig # 生成 .config 文件
make prepare
make -j $(nproc)
```

```shell
#!/bin/sh

kernel="workdir/build/sqlite_qemu-x86_64"
cmd=""

if test $# -eq 1; then
    kernel="$1"
fi

rootfs="./rootfs"
test -d "$rootfs" || mkdir "$rootfs"

# Clean up any previous instances.
sudo pkill -f qemu-system
sudo pkill -f firecracker
sudo kraft stop --all
sudo kraft rm --all

# Create CPIO archive to be used as the initrd.
old="$PWD"
cd "$rootfs"
find -depth -print | tac | bsdcpio -o --format newc > "$old"/rootfs.cpio
cd "$old"

qemu-system-x86_64 \
    -kernel "$kernel" \
    -nographic \
    -m 64M \
    -append "$cmd" \
    -initrd "$PWD"/rootfs.cpio \
    -cpu max
```