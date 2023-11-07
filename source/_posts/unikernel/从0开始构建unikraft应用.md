---
title: 从 0 开始构建 unikraft 应用
tags: unikernel
abbrlink: e12ad5d
date: 2023-10-25 18:30:13
---
本文档将从空函数，helloworld，redis 三个例子构建 unikraft qemu-x6_64 应用。
<!-- more --> 
## 从空函数开始
首先从最简单的例子开始，要运行一个空函数，即 main.c 里面的内容为：
```c
int main() {
    return 0;
}
```
首先创建 Makefile 文件，Makefile 文件的内容参考 [Unikraft 安装与使用](https://peizhongqiu.github.io/p/8e98bdb9.html)：
```makefile
UK_ROOT ?= $(PWD)/workdir/unikraft
UK_LIBS ?= $(PWD)/workdir/libs
UK_BUILD ?= $(PWD)/workdir/build
LIBS :=

all:
	@$(MAKE) -C $(UK_ROOT) A=$(PWD) L=$(LIBS) O=$(UK_BUILD)

$(MAKECMDGOALS):
	@$(MAKE) -C $(UK_ROOT) A=$(PWD) L=$(LIBS) O=$(UK_BUILD) $(MAKECMDGOALS)
```

然后创建 Makefile.uk，由于创建的 app 的名字为 none，所以参数为 appnone，APPNONE_*：
```c
$(eval $(call addlib,appnone))

APPNONE_SRCS-y += $(APPNONE_BASE)/main.c
```

接着，创建一个 Config.uk 文件。由于运行的是一个空函数，所以 Config.uk 里面可以不存放任何内容。

然后，创建一个 defconfigs 文件夹，里面创建 qemu-x86_64 文件，里面包含运行应用需要的最小配置：
```config
CONFIG_PLAT_KVM=y
CONFIG_LIBUKBUS=y
CONFIG_LIBUKSGLIST=y
CONFIG_UK_DEFNAME="none"
CONFIG_UK_NAME="none"
```
最后，创建 scripts 文件夹，里面创建 build 和 run 文件夹，分别存放构建和运行脚本。在 build 里面创建 make-qemu-x86_64.sh：
```shell
#!/bin/sh

make distclean
UK_DEFCONFIG=$(pwd)/defconfigs/qemu-x86_64 make defconfig # 生成 .config 文件
make prepare
make -j $(nproc)
```
在 run 里面创建 qemu-x86_64.sh：
```shell
#!/bin/sh

kernel="workdir/build/none_qemu-x86_64"

if test $# -eq 1; then
    kernel="$1"
fi

# Clean up any previous instances.
sudo pkill -f qemu-system
sudo pkill -f firecracker
sudo kraft stop --all
sudo kraft rm --all

qemu-system-x86_64 \
    -kernel "$kernel" \
    -nographic \
    -m 64M 
```
在 scripts 里创建 setup.sh 用来下载 unikraft 和相关库。
```shell
#!/bin/bash

test ! -d "workdir" && echo "Cloning repositories..." || true
test ! -d "workdir/unikraft" && git clone https://github.com/unikraft/unikraft workdir/unikraft || true
```
此时，我们就可以运行 none 程序了。首先使用以下命令下载，构建和运行：
```shell
./script/setup.sh
./script/build/make-qemu-x86-64.sh
./script/run/qemu-x86-64.sh 
```
运行结果如下图所示：
{% asset_img  none.png 图 1 none%}

在前面我们提到构建运行的方法除了 make 以外，还能通过 kraft 实现，所以需要创建 kraft.yaml 文件：
```yaml
specification: '0.5'
name: none
unikraft:
  version: stable
targets:
  - name: none-linuxu-x86_64
    architecture: x86_64
    platform: linuxu
  - name: none-qemu-x86_64
    architecture: x86_64
    platform: qemu
```
创建完之后，就可以通过 kraft 工具构建和运行：
```shell
kraft build --log-level debug --log-type basic --target none-qemu-x86_64 --plat qemu
kraft run -W --memory 64M --log-level debug --log-type basic --target none-qemu-x86_64 --plat qemu 
```
也可以在 scripts 文件夹中的 build 和 run 文件夹添加 kraft 运行脚本。
在 build 里面创建 kraft-qemu-x86_64.sh：
``` shell
#!/bin/sh

rm -fr .unikraft
rm -f .config.*
kraft build --log-level debug --log-type basic --target none-qemu-x86_64 --plat qemu
```
在 run 里面创建 kraft-qemu-x86_64.sh：
```shell
#!/bin/sh

# Clean up any previous instances.
sudo pkill -f qemu-system
sudo pkill -f firecracker
sudo kraft stop --all
sudo kraft rm --all

kraft run \
    -W \
    --memory 64M \
    --log-level debug --log-type basic \
    --target none-qemu-x86_64 --plat qemu 
```
运行结果和上图一致。

最后，所有添加的文件目录如下所示：
```shell
.
├── Config.uk
├── defconfigs
│   └── qemu-x86_64
├── kraft.yaml
├── main.c
├── Makefile
├── Makefile.uk
└── scripts
    ├── build
    │   ├── kraft-qemu-x86-64.sh
    │   └── make-qemu-x86-64.sh
    ├── run
    │   ├── kraft-qemu-x86-64.sh
    │   └── qemu-x86-64.sh
    └── setup.sh
```
## helloworld
代码来自于：https://github.com/unikraft/app-helloworld。
```c
#include <stdio.h>

/* Import user configuration: */
#ifdef __Unikraft__
#include <uk/config.h>
#endif /* __Unikraft__ */

#if CONFIG_APPHELLOWORLD_SPINNER
#include <time.h>
#include <errno.h>
#include "monkey.h"

static void millisleep(unsigned int millisec)
{
	struct timespec ts;
	int ret;

	ts.tv_sec = millisec / 1000;
	ts.tv_nsec = (millisec % 1000) * 1000000;
	do
		ret = nanosleep(&ts, &ts);
	while (ret && errno == EINTR);
}
#endif /* CONFIG_APPHELLOWORLD_SPINNER */

int main(int argc, char *argv[])
{
#if CONFIG_APPHELLOWORLD_PRINTARGS || CONFIG_APPHELLOWORLD_SPINNER
	int i;
#endif

	printf("Hello world!\n");

#if CONFIG_APPHELLOWORLD_PRINTARGS
	printf("Arguments: ");
	for (i=0; i<argc; ++i)
		printf(" \"%s\"", argv[i]);
	printf("\n");
#endif /* CONFIG_APPHELLOWORLD_PRINTARGS */

#if CONFIG_APPHELLOWORLD_SPINNER
	i = 0;
	printf("\n\n\n");
	for (;;) {
		i %= (monkey3_frame_count * 3);
		printf("\r\033[2A %s \n", monkey3[i++]);
		printf(" %s \n",          monkey3[i++]);
		printf(" %s ",            monkey3[i++]);
		fflush(stdout);
		millisleep(250);
	}
#endif /* CONFIG_APPHELLOWORLD_SPINNER */

	return 0;
}
```
monkey.h 是代码里的一个头文件，用来显示猴子图案。
与刚才的空函数相比，感觉丰富了一点。可以看到，这里面有两个 config 变量：CONFIG_APPHELLOWORLD_SPINNER，CONFIG_APPHELLOWORLD_PRINTARGS。因此，需要往 Config.uk 里面添加：
```config
### Invisible option for dependencies
config APPHELLOWORLD_DEPENDENCIES
	bool
	default y
	select LIBNOLIBC if !HAVE_LIBC

### App configuration
config APPHELLOWORLD_PRINTARGS
	bool "Print arguments"
	default y
	help
	  Prints argument list (argv) to stdout

config APPHELLOWORLD_SPINNER
	bool "Stay alive"
	select LIBPOSIX_TIME
	default n
	help
	  Shows an animation instead of shutting down
```
通过 Config.uk 可以看到 helloworld 这个程序当 HAVE_LIBC 没有声明时，就会把 LIBNOLIBC 置 y。APPHELLOWORLD_PRINTARGS 默认值为 y，APPHELLOWORLD_SPINNER 默认值为 n。

虽然这个程序看起来比空程序更复杂，但是需要的时钟和系统调用在 unikraft 的微库中都已经实现，所以其他部分的内容与空程序基本一致。需要修改以下文件：
defconfigs/qemu-x86_64
```config
CONFIG_PLAT_KVM=y
CONFIG_LIBUKBUS=y
CONFIG_LIBUKSGLIST=y
CONFIG_UK_DEFNAME="helloworld"
CONFIG_UK_NAME="helloworld"
```
scripts/run/qemu-x86-64.sh
```shell
#!/bin/sh

kernel="workdir/build/helloworld_qemu-x86_64"

if test $# -eq 1; then
    kernel="$1"
fi

# Clean up any previous instances.
sudo pkill -f qemu-system
sudo pkill -f firecracker
sudo kraft stop --all
sudo kraft rm --all

qemu-system-x86_64 \
    -kernel "$kernel" \
    -nographic \
    -m 64M 
```
Makefile.uk
```makefile
$(eval $(call addlib,apphelloworld))

APPHELLOWORLD_SRCS-y += $(APPHELLOWORLD_BASE)/main.c
```

运行结果如下图所示：
{% asset_img  hello.png 图 2 hello1%}
由于 APPHELLOWORLD_SPINNER 默认为 n，所以最后一部分 monkey 代码没有打印。所以，可以通过 make menuconfig 指令打开。运行结果如下图所示：
{% asset_img  hello2.png 图 3 hello2%}

## redis
以上的内容都比较简单。接下来，我们看一下复杂的应用程序 redis 该如何移植到 unikraft 上。
首先，由于 redis 应用程序庞大，为了以后方便能够移植到其他 unikrat 镜像中，我们将 redis 分成两部分：App 和 Lib。

### App
App 部分较为简单。首先需要一个 Makefile 文件。Makefile 文件的内容为：
```shell
UK_ROOT ?= $(PWD)/workdir/unikraft
UK_LIBS ?= $(PWD)/workdir/libs
UK_BUILD ?= $(PWD)/workdir/build
LIBS := $(UK_LIBS)/musl:$(UK_LIBS)/lwip:$(UK_LIBS)/redis

all:
	@$(MAKE) -C $(UK_ROOT) A=$(PWD) L=$(LIBS) O=$(UK_BUILD)

$(MAKECMDGOALS):
	@$(MAKE) -C $(UK_ROOT) A=$(PWD) L=$(LIBS) O=$(UK_BUILD) $(MAKECMDGOALS)
```
与 none 和 helloworld 例子不同的是，redis 依赖一些库：musl，lwip 和 libredis。musl 是一个轻量级的 c 库，lwip 是一个轻量级 TCP/IP 协议。这两个在 unikraft 的 github 中都已经完成移植，我们直接拿来使用，我们的主要任务是完成 libredis 的移植。

之后创建 Makefile.uk 文件：
```makefile
$(eval $(call addlib,appredis))
```
这个文件比较简单。与 none 和 helloworld 不同的是，在这个 appredis 里面没有其他源文件，所有相关的源文件都放到了 libredis 中，所以不需要添加其他内容。

之后，创建 defconfigs/qemu-x86_64-9pfs，向其中写入：
```config
CONFIG_UK_NAME="redis"
CONFIG_UK_DEFNAME="redis"
CONFIG_PLAT_KVM=y
CONFIG_LIBDEVFS_AUTOMOUNT=y
CONFIG_LIBUKBUS=y
CONFIG_LIBUKDEBUG_REDIR_PRINTK=y
CONFIG_LIBUKFALLOCBUDDY=y
CONFIG_LIBVFSCORE_AUTOMOUNT_ROOTFS=y
CONFIG_LIBVFSCORE_ROOTFS_9PFS=y
CONFIG_LIBVFSCORE_ROOTDEV="fs1"
CONFIG_LIBVFSCORE_ROOTOPTS="version=9P2000.L"
CONFIG_LWIP_DHCP=y
CONFIG_LIBREDIS=y
CONFIG_LIBREDIS_SERVER_MAIN_FUNCTION=y
```
同样的，可以通过 make menuconfig 来选择需要的配置。

最后，我们需要创建 scripts/build，scripts/run 文件夹。创建构建和运行脚本。

scripts/build/make-qemu-x86_64-9pfs
```shell
#!/bin/sh

make distclean
UK_DEFCONFIG=$(pwd)/defconfigs/qemu-x86_64-9pfs make defconfig V=1
make prepare V=1
make -j $(nproc) V=1
```
scripts/run/qemu-x86_64-9pfs
```shell
#!/bin/sh

kernel="workdir/build/redis_qemu-x86_64"
cmd="/redis.conf"

if test $# -eq 1; then
    kernel="$1"
fi

rootfs="rootfs/"
test -d "$rootfs" || mkdir "$rootfs"

# Clean up any previous instances.
sudo pkill -f qemu-system
sudo pkill -f firecracker
sudo kraft stop --all
sudo kraft rm --all

# Remove previously created network interfaces.
sudo ip link set dev tap0 down 2> /dev/null
sudo ip link del dev tap0 2> /dev/null
sudo ip link set dev virbr0 down 2> /dev/null
sudo ip link del dev virbr0 2> /dev/null

# Create bridge interface for QEMU networking.
sudo ip link add dev virbr0 type bridge
sudo ip address add 172.44.0.1/24 dev virbr0
sudo ip link set dev virbr0 up

sudo qemu-system-x86_64 \
    -kernel "$kernel" \
    -nographic \
    -m 256M \
    -netdev bridge,id=en0,br=virbr0 -device virtio-net-pci,netdev=en0 \
    -append "netdev.ipv4_addr=172.44.0.2 netdev.ipv4_gw_addr=172.44.0.1 netdev.ipv4_subnet_mask=255.255.255.0 -- $cmd" \
    -fsdev local,id=myid,path="$rootfs",security_model=none \
    -device virtio-9p-pci,fsdev=myid,mount_tag=fs1,disable-modern=on,disable-legacy=off \
    -cpu max
```
注意，build 和 run 脚本可以自动生成。生成的源文件：
```shell
wget https://raw.githubusercontent.com/unikraft/app-testing/staging/scripts/generate.py -O scripts/generate.py
```
generate.py 运行需要 run.yaml：
```yaml
runs:
  - rootfs: rootfs/
    memory: 256
    networking: True
    command: /redis.conf
```
最后配置 redis 运行需要的配置文件 rootfs/redis.conf:
```config
bind 0.0.0.0
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
protected-mode no
```
这样，app 部分就顺利完成了。

### Lib
接下来，需要完成库函数部分。首先创建 workdir 目录。将 unikraft 仓库源代码放入 workdir/unikraft 目录下。在 workdir 目录下创建 libs 目录。将别人移植好的 musl 和 lwip 仓库拷贝进 workdir/libs 文件夹下。这部分与 app 部分 makefile 文件的配置相对应。接着，开始移植 libredis。

首先，在 workdir/libs 下创建 redis 目录。将 redis 7.0.14 版本代码下载到 redis/src 目录下。
在 redis 目录下创建 Config.uk 文件（代码来源于 https://github.com/unikraft/lib-redis/blob/staging/Config.uk）：
```config
menuconfig LIBREDIS
	bool "Redis"
	default n

if LIBREDIS
# hidden
config LIBREDIS_COMMON
	bool
	default n
	select LIBUKDEBUG
	select LIBUKALLOC
	select LIBUKSCHED
	select LIBNEWLIBC
	select LIBPOSIX_EVENT
	select LIBNEWLIBC_WANT_IO_C99_FORMATS if LIBNEWLIBC
	select LIBNEWLIBC_LINUX_ERRNO_EXTENSIONS if LIBNEWLIBC
	select LIBPTHREAD_EMBEDDED
	select LIBPOSIX_SYSINFO
	select LIBPOSIX_LIBDL
	select LIBPOSIX_SOCKET
	select LIBPOSIX_EVENT
	select LIBLWIP
	select LWIP_IPV6
	select LWIP_TCP_KEEPALIVE
	select LWIP_TCP_TIMESTAMPS
	select LIBUKSWRAND_DEVFS
	select LIBMUSL
# hidden
config LIBREDIS_MAIN_FUNCTION
	bool
	default n

config LIBREDIS_SERVER
	bool "Redis server"
	default y
	select LIBREDIS_COMMON
	select LIBREDIS_HISTOGRAM
	select LIBREDIS_HIREDIS
	imply LIBREDIS_LUA
	help
		Build the Redis server library.

	if LIBREDIS_SERVER
		config LIBREDIS_SERVER_MAIN_FUNCTION
			bool "Provide main function"
			default n
			select LIBREDIS_MAIN_FUNCTION
	endif

config LIBREDIS_CLIENT
	bool "Redis client"
	default n
	select LIBREDIS_COMMON
	select LIBREDIS_HISTOGRAM
	select LIBREDIS_HIREDIS
	help
		Build the Redis client library.

	if LIBREDIS_CLIENT
		config LIBREDIS_CLIENT_MAIN_FUNCTION
			bool "Provide main function"
			default n
			select LIBREDIS_MAIN_FUNCTION
	endif

config LIBREDIS_LUA
	bool "Use internal Lua implementation"
	default n

config LIBREDIS_HIREDIS
	bool "Use internal Hiredis implementation"
	default n

config LIBREDIS_HISTOGRAM
	bool "Use internal Histogram implementation"
	default n
endif
```
创建 Makefile.uk，此部分代码通过修改 https://github.com/unikraft/lib-redis/blob/staging/Makefile.uk 得到。
第一步，调用 Unikraft addlib 函数向系统注册应用程序。这里 lua，hiredis，histogram 是 redis server 的依赖项。
```makefile
################################################################################
# Library registration
################################################################################
$(eval $(call addlib_s,libredis,$(CONFIG_LIBREDIS)))
$(eval $(call addlib_s,libredis_lua,$(CONFIG_LIBREDIS_LUA)))
$(eval $(call addlib_s,libredis_hiredis,$(CONFIG_LIBREDIS_HIREDIS)))
$(eval $(call addlib_s,libredis_histogram,$(CONFIG_LIBREDIS_HISTOGRAM)))
$(eval $(call addlib_s,libredis_common,$(CONFIG_LIBREDIS)))
```
第二步，声明源代码的位置。
```makefile
################################################################################
# Sources
################################################################################
LIBREDIS_VERSION=7.0.14
LIBREDIS_BASENAME=redis-$(LIBREDIS_VERSION)
LIBREDIS_SRCDIR=$(LIBREDIS_BASE)/src

################################################################################
# Helpers
################################################################################
LIBREDIS_SRC  = $(LIBREDIS_SRCDIR)/$(LIBREDIS_BASENAME)/src
LIBREDIS_DEPS = $(LIBREDIS_SRCDIR)/$(LIBREDIS_BASENAME)/deps

################################################################################
# Library includes
################################################################################

LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_HIREDIS) += -I$(LIBREDIS_DEPS)/hiredis
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_HISTOGRAM) += -I$(LIBREDIS_DEPS)/hdr_histogram
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_LUA) += -I$(LIBREDIS_DEPS)/lua/src
```
第三步，添加源文件和编译选项。注意，并不是所有的源文件都需要添加，为了精简镜像，应当把不需要的源文件删除。
```makefile
################################################################################
# Flags
################################################################################


################################################################################
# Redis internal Lua
################################################################################
LIBREDIS_LUA_CFLAGS-y += $(LIBREDIS_CFLAGS-y)
LIBREDIS_LUA_CFLAGS-y += -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''

LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/fpconv.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lapi.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lauxlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lbaselib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lcode.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldblib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldebug.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldo.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldump.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lfunc.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lgc.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/linit.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/liolib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/llex.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lmathlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lmem.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/loadlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lobject.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lopcodes.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/loslib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lparser.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstate.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstring.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstrlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltable.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltablib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltm.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_bit.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_cjson.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_cmsgpack.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_struct.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/luac.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lundump.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lvm.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lzio.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/print.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/strbuf.c

LIBREDIS_LUA_LUA_CJSON_FLAGS-y += -Wno-sign-compare
LIBREDIS_LUA_LUA_CMSGPACK_FLAGS-y += -Wno-sign-compare

################################################################################
# Redis internal Hiredis
################################################################################
LIBREDIS_HIREDIS_CFLAGS-y += $(LIBREDIS_CFLAGS-y)

LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/alloc.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/async.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/dict.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/hiredis.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/net.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/read.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/sds.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/ssl.c

################################################################################
# Redis internal hdr histogram
################################################################################
LIBREDIS_HISTOGRAM_CFLAGS-y += $(LIBREDIS_CFLAGS-y)

LIBREDIS_HISTOGRAM_SRCS-y += $(LIBREDIS_DEPS)/hdr_histogram/hdr_histogram.c

################################################################################
# Functionality shared between server and client
################################################################################
LIBREDIS_CFLAGS-y    += -Wno-missing-field-initializers
LIBREDIS_CFLAGS-y    += -DREDIS_STATIC=''

LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/acl.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/adlist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/ae.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/anet.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/aof.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/bio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/bitops.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/blocked.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/call_reply.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/childinfo.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/cluster.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/commands.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/config.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/connection.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crc16.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crc64.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crcspeed.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/db.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/debug.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/defrag.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/dict.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/endianconv.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/eval.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/evict.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/expire.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/function_lua.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/functions.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geo.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geohash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geohash_helper.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/hyperloglog.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/intset.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/latency.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lazyfree.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/listpack.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/localtime.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut5.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut6.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lzf_c.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lzf_d.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/memtest.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/module.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/monotonic.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/mt19937-64.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/multi.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/networking.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/notify.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/object.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/pqsort.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/pubsub.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/quicklist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rand.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rax.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rdb.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/redis-check-aof.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/redis-check-rdb.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/release.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/replication.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/resp_parser.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/script.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/script_lua.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sds.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sentinel.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/setcpuaffinity.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/setproctitle.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sha1.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sha256.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/siphash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/slowlog.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sort.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sparkline.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/syncio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/syscheck.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_hash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/timeout.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_list.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/tls.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/tracking.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_set.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_stream.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_string.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_zset.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/util.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/ziplist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/zipmap.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/zmalloc.c

LIBREDIS_SRCS-$(CONFIG_LIBREDIS_MAIN_FUNCTION) += $(LIBREDIS_BASE)/main.c|unikraft

################################################################################
# Redis server
################################################################################
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/server.c
LIBREDIS_SERVER_FLAGS-y += -Dmain=redis_server_main

################################################################################
# Redis client
################################################################################
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_CLIENT) += -I$(LIBREDIS_DEPS)/linenoise

LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_SRC)/cli_common.c
LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_DEPS)/linenoise/linenoise.c
LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_SRC)/redis-cli.c

LIBREDIS_REDIS-CLI_FLAGS-y += -Dmain=redis_client_main
```

此时运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本会报错：
```shell
In file included from /home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/src/server.h:37,
                 from /home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/src/acl.c:30:
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/src/atomicvar.h:91:10: fatal error: stdatomic.h: No such file or directory
   91 | #include <stdatomic.h>
      |          ^~~~~~~~~~~~~
compilation terminated.
```
我们需要在第三步的 Flags 部分添加代码：
```makefile
################################################################################
# Flags
################################################################################
LIBREDIS_FLAGS_SUPPRESS = -D__STDC_NO_ATOMICS__
LIBREDIS_CFLAGS-y += $(LIBREDIS_FLAGS_SUPPRESS)
```
但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/src/release.c:37:10: fatal error: release.h: No such file or directory
   37 | #include "release.h"
      |          ^~~~~~~~~~~
compilation terminated.
```
这个 release.h 是 redis 在 make 阶段通过 mkreleasehdr.sh 脚本自动生成的，所以我们需要自己写一个 release.h。在 libs/redis 下新建 include 目录，并添加 release.h 文件：
```c
#define REDIS_GIT_SHA1 "ffffffff"
#define REDIS_GIT_DIRTY "86"
#define REDIS_BUILD_ID "xxxxxxxx"
```
其中的值可以自由填写。
之后在 Makefile.uk 中第二步 Library includes 添加以下代码：
```makefile
################################################################################
# Library includes
################################################################################
CINCLUDES-y   += -I$(LIBREDIS_BASE)/include
CXXINCLUDES-y += -I$(LIBREDIS_BASE)/include
```
但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/src/setcpuaffinity.c:90:15: error: ‘cpuset’ undeclared (first use in this function)
   90 |     CPU_ZERO(&cpuset);
      |               ^~~~~~
```
这里，我们需要把 setcpuaffinity.c 文件里面的 setcpuaffinity 函数清空。
但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/deps/hdr_histogram/hdr_histogram.c:21:28: fatal error: hdr_malloc.h: No such file or y
   21 | #define HDR_MALLOC_INCLUDE "hdr_malloc.h"
      |                            ^~~~~~~~~~~~~~
compilation terminated.
```
这是因为编译 hdr histogram 参数需要传递 -DHDR_MALLOC_INCLUDE 参数。
```makefile
################################################################################
# Redis internal hdr histogram
################################################################################
LIBREDIS_HISTOGRAM_CFLAGS-y += $(LIBREDIS_CFLAGS-y)
LIBREDIS_HISTOGRAM_CFLAGS-y += -DHDR_MALLOC_INCLUDE=\"hdr_redis_malloc.h\"
```
但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/deps/hiredis/ssl.c:46:10: fatal error: openssl/ssl.h: No such file or directory
   46 | #include <openssl/ssl.h>
      |          ^~~~~~~~~~~~~~~
compilation terminated.
```
这是因为没有安装 openssl，由于 ssl.c 是在依赖 hiredis 内部，且与 redis 无关，所以，需要把 ssl.c 文件删除。

但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/usr/local/bin/ld: /home/wark/unikraft/apps/pre/redis/workdir/build/libredis_lua/luac.o: in function `main':
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/deps/lua/src/luac.c:187: multiple definition of `main'; 
```
查看 deps/lua/src/luac.c，发现有 main 函数，所以需要把 deps/lua/src/luac.c 从源文件中删除。
但是，运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本依然会报错：
```shell
/usr/local/bin/ld: /home/wark/unikraft/apps/pre/redis/workdir/build/libredis_lua.o: in function `main':
/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/src/redis-7.0.14/deps/lua/src/lua.c:377: multiple definition of `main'; /home/wark/unikraft/apps/pre/redis/workdir/build/libredis.o:/home/wark/unikraft/apps/pre/redis/workdir/libs/redis/main.c:80: first defined here
```
查看 deps/lua/src/lua.c，发现有 main 函数，所以需要把 deps/lua/src/lua.c 从源文件中删除。

通过这样的改进，现在运行 redis/scripts/build/make-qemu-x86_64-9pfs.sh 的构建脚本就不存在问题。
最终 makefile.uk 内容：
```makefile
################################################################################
# Library registration
################################################################################
$(eval $(call addlib_s,libredis,$(CONFIG_LIBREDIS)))
$(eval $(call addlib_s,libredis_lua,$(CONFIG_LIBREDIS_LUA)))
$(eval $(call addlib_s,libredis_hiredis,$(CONFIG_LIBREDIS_HIREDIS)))
$(eval $(call addlib_s,libredis_histogram,$(CONFIG_LIBREDIS_HISTOGRAM)))
$(eval $(call addlib_s,libredis_common,$(CONFIG_LIBREDIS)))

################################################################################
# Sources
################################################################################
LIBREDIS_VERSION=7.0.14
LIBREDIS_BASENAME=redis-$(LIBREDIS_VERSION)
LIBREDIS_SRCDIR=$(LIBREDIS_BASE)/src

################################################################################
# Helpers
################################################################################
LIBREDIS_SRC  = $(LIBREDIS_SRCDIR)/$(LIBREDIS_BASENAME)/src
LIBREDIS_DEPS = $(LIBREDIS_SRCDIR)/$(LIBREDIS_BASENAME)/deps

################################################################################
# Library includes
################################################################################
CINCLUDES-y   += -I$(LIBREDIS_BASE)/include
CXXINCLUDES-y += -I$(LIBREDIS_BASE)/include

LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_HIREDIS) += -I$(LIBREDIS_DEPS)/hiredis
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_HISTOGRAM) += -I$(LIBREDIS_DEPS)/hdr_histogram
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_LUA) += -I$(LIBREDIS_DEPS)/lua/src

################################################################################
# Flags
################################################################################
# Suppress some warnings to make the build process look neater
LIBREDIS_FLAGS_SUPPRESS = -D__STDC_NO_ATOMICS__
LIBREDIS_CFLAGS-y += $(LIBREDIS_FLAGS_SUPPRESS)

################################################################################
# Redis internal Lua
################################################################################
LIBREDIS_LUA_CFLAGS-y += $(LIBREDIS_CFLAGS-y)
LIBREDIS_LUA_CFLAGS-y += -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''

LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/fpconv.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lapi.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lauxlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lbaselib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lcode.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldblib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldebug.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldo.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ldump.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lfunc.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lgc.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/linit.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/liolib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/llex.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lmathlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lmem.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/loadlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lobject.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lopcodes.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/loslib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lparser.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstate.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstring.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lstrlib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltable.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltablib.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/ltm.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_bit.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_cjson.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_cmsgpack.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lua_struct.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lundump.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lvm.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/lzio.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/print.c
LIBREDIS_LUA_SRCS-y += $(LIBREDIS_DEPS)/lua/src/strbuf.c

LIBREDIS_LUA_LUA_CJSON_FLAGS-y += -Wno-sign-compare
LIBREDIS_LUA_LUA_CMSGPACK_FLAGS-y += -Wno-sign-compare

################################################################################
# Redis internal Hiredis
################################################################################
LIBREDIS_HIREDIS_CFLAGS-y += $(LIBREDIS_CFLAGS-y)

LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/alloc.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/async.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/dict.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/hiredis.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/net.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/read.c
LIBREDIS_HIREDIS_SRCS-y += $(LIBREDIS_DEPS)/hiredis/sds.c

################################################################################
# Redis internal hdr histogram
################################################################################
LIBREDIS_HISTOGRAM_CFLAGS-y += $(LIBREDIS_CFLAGS-y)
LIBREDIS_HISTOGRAM_CFLAGS-y += -DHDR_MALLOC_INCLUDE=\"hdr_redis_malloc.h\"

LIBREDIS_HISTOGRAM_SRCS-y += $(LIBREDIS_DEPS)/hdr_histogram/hdr_histogram.c

################################################################################
# Functionality shared between server and client
################################################################################
LIBREDIS_CFLAGS-y    += -Wno-missing-field-initializers
LIBREDIS_CFLAGS-y    += -DREDIS_STATIC=''

LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/acl.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/adlist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/ae.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/anet.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/aof.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/bio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/bitops.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/blocked.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/call_reply.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/childinfo.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/cluster.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/commands.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/config.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/connection.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crc16.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crc64.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/crcspeed.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/db.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/debug.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/defrag.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/dict.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/endianconv.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/eval.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/evict.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/expire.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/function_lua.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/functions.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geo.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geohash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/geohash_helper.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/hyperloglog.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/intset.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/latency.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lazyfree.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/listpack.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/localtime.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut5.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut6.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lolwut.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lzf_c.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/lzf_d.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/memtest.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/module.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/monotonic.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/mt19937-64.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/multi.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/networking.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/notify.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/object.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/pqsort.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/pubsub.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/quicklist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rand.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rax.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rdb.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/redis-check-aof.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/redis-check-rdb.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/release.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/replication.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/resp_parser.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/rio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/script.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/script_lua.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sds.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sentinel.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/setcpuaffinity.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/setproctitle.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sha1.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sha256.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/siphash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/slowlog.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sort.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/sparkline.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/syncio.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/syscheck.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_hash.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/timeout.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_list.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/tls.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/tracking.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_set.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_stream.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_string.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/t_zset.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/util.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/ziplist.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/zipmap.c
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/zmalloc.c

LIBREDIS_SRCS-$(CONFIG_LIBREDIS_MAIN_FUNCTION) += $(LIBREDIS_BASE)/main.c|unikraft

################################################################################
# Redis server
################################################################################
LIBREDIS_SRCS-y += $(LIBREDIS_SRC)/server.c
LIBREDIS_SERVER_FLAGS-y += -Dmain=redis_server_main

################################################################################
# Redis client
################################################################################
LIBREDIS_CINCLUDES-$(CONFIG_LIBREDIS_CLIENT) += -I$(LIBREDIS_DEPS)/linenoise

LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_SRC)/cli_common.c
LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_DEPS)/linenoise/linenoise.c
LIBREDIS_SRCS-$(CONFIG_LIBREDIS_CLIENT) += $(LIBREDIS_SRC)/redis-cli.c

LIBREDIS_REDIS-CLI_FLAGS-y += -Dmain=redis_client_main
```

接下来运行 redis。
```shell
SeaBIOS (version 1.15.0-1)


iPXE (https://ipxe.org) 00:03.0 CA00 PCI2.10 PnP PMM+0FF8B2F0+0FECB2F0 CA00
                                                                               


Booting from ROM..1: Set IPv4 address 172.44.0.2 mask 255.255.255.0 gw 172.44.0.1
en1: Added
en1: Interface is up
Powered by
o.   .o       _ _               __ _
Oo   Oo  ___ (_) | __ __  __ _ ' _) :_
oO   oO ' _ `| | |/ /  _)' _` | |_|  _)
oOo oOO| | | | |   (| | | (_) |  _) :_
 OoOoO ._, ._:_:_,\_._,  .__,_:_, \___)
                Pandora 0.15.0~97523b55
1:C 14 Aug 2015 08:05:51.049 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 14 Aug 2015 08:05:51.051 # Redis version=7.0.14, bits=64, commit=ffffffff, modified=1, pid=1, just started
1:C 14 Aug 2015 08:05:51.051 # Configuration loaded
[    0.163579] ERR:  [libposix_process] <deprecated.c @  348> Ignore updating resource 7: cur = 10032, max = 10032
1:M 14 Aug 2015 08:05:51.064 * Increased maximum number of open files to 10032 (it was originally set to 1024).
1:M 14 Aug 2015 08:05:51.064 * monotonic clock: POSIX clock_gettime
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 7.0.14 (ffffffff/1) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                  
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           https://redis.io       
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

1:M 14 Aug 2015 08:05:51.094 # WARNING: The TCP backlog setting of 511 cannot be enforced because SOMAXCONN is set to the lower value of 128.
1:M 14 Aug 2015 08:05:51.095 # Server initialized
1:M 14 Aug 2015 08:05:51.103 * Ready to accept connections
```
在服务器上打开另一个终端，输入以下命令：
```shell
redis-cli -h 172.44.0.2
172.44.0.2:6379> set a 1
OK
172.44.0.2:6379> get a
"1"
172.44.0.2:6379> 
```
这样 redis 完成移植。
最终 APP 目录下有以下文件：
```shell
.
├── defconfigs
│   └── qemu-x86_64-9pfs
├── Makefile
├── Makefile.uk
├── rootfs
│   └── redis.conf
├── scripts
│   ├── build
│   │   └── make-qemu-x86_64-9pfs.sh
│   ├── run
│   │   └── qemu-x86_64-9pfs.sh
│   └── setup.sh
└── workdir
    ├── libs
    │   ├── lwip
    │   ├── musl
    │   └── redis
    └── unikraft
```
最终 libs/redis 目录下有以下文件：
```shell
.
├── Config.uk
├── include
│   └── release.h
├── main.c
├── Makefile.uk
├── patches
└── src
    ├── 7.0.14.zip
    └── redis-7.0.14
        ├── 00-RELEASENOTES
        ├── BUGS
        ├── CODE_OF_CONDUCT.md
        ├── CONTRIBUTING.md
        ├── COPYING
        ├── deps
        ├── INSTALL
        ├── Makefile
        ├── MANIFESTO
        ├── README.md
        ├── redis.conf
        ├── runtest
        ├── runtest-cluster
        ├── runtest-moduleapi
        ├── runtest-sentinel
        ├── SECURITY.md
        ├── sentinel.conf
        ├── src
        ├── tests
        ├── TLS.md
        └── utils
```