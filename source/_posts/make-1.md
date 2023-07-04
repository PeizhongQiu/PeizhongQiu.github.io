---
title: Makefiles 简介
tags: makefile
abbrlink: c677bb8a
date: 2022-04-20 14:30:56
---

内容摘自《GNU Make》第二章。

本文将讨论一个简单的 Makefile，这个 Makefile 将描述如何编译和连接 8 个 C 语言源代码文件和 3 个头文件。当 make 重新编译，每一个修改了的 C 语言源代码文件都要重新编译。如果头文件发生了修改，所有包含了该头文件的源代码文件要重新编译。每一次编译都会产生一个目标文件，当发生重新编译时，所有文件都要重新链接成一个新的可执行文件。

## 规则

一个简单的 Makefile 由很多规则组成，规则有着以下的模式：

```makefile
target ... : prerequisites ...
	recipe
	...
	...
```

target 一般是程序要生成的文件名字，比如说是可执行文件或目标文件的名字。target 还可能是需要执行的操作的名字，比如 clean。

prerequisites 是被用作创建 target 的依赖文件。target 一般会依赖一些文件。

recipe 是 make 要执行的操作。recipe 可以有超过一条命令。注意，**必须在每个 recipe 行的开始加上 tab。**recipe 一般在有 prerequisites 的规则中，当 prerequisites 中的文件变化时创建 target 文件。但是，一些规则指定了 recipe 就不需要 prerequisites，比如 clean。

## 简单的例子

```
edit : main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
		cc -o edit main.o kbd.o command.o display.o \
					insert.o search.o files.o utils.o
					
main.o : main.c defs.h
		cc -c main.c
kbd.o : kbd.c defs.h command.h
		cc -c kbd.c
command.o : command.c defs.h command.h
		cc -c command.c
display.o : display.c defs.h buffer.h
		cc -c display.c
insert.o : insert.c defs.h buffer.h
		cc -c insert.c
search.o : search.c defs.h buffer.h
		cc -c search.c
files.o : files.c defs.h buffer.h command.h
		cc -c files.c
utils.o : utils.c defs.h
		cc -c utils.c
		
clean :
		rm edit main.o kbd.o command.o display.o \
			insert.o search.o files.o utils.o
```

如果想要创建名为 edit 的可执行文件：

```
make
```

如果想要删除所有目录下的所有可执行文件和目标文件：

```
make clean
```

## make 执行 Makefile

在默认情况下，make 从第一个 target 开始执行（不是以 ”.“ 开头 target）。这个被称为 default goal。Goal 是 make 努力更新的最终 targets。上面的例子中，default goal 就是edit，所以是第一个规则。

在执行 make 时，首先读取当前目录下的 makefile 文件然后处理第一个规则。在上面的例子中，这个规则是用来链接 edit，但是在 make 能完全处理这个规则之前，make 必须处理 edit 依赖的文件的规则，在上面这个例子中就是目标文件。每个目标文件都要根据自己的规则进行处理。

如果一个规则不被 goal 依赖，则这个命令就不会被处理，除非你告诉 make 要执行，比如make clean。

当目标文件不存在，或源文件，头文件比目标文件更新时，会进行重新编译。当执行文件不存在，或者执行文件比目标文件更晚时，执行文件会更新。

## 变量

在上一个例子中，我们在 edit 规则中枚举所有的目标文件两次，这是很不方便的。如果有新的目标文件加入，可能会导致忘记在所有地方都修改。因此可以使用变量简化。

```
objects = main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
		
edit : $(objects)
		cc -o edit $(objects)
		
main.o : main.c defs.h
		cc -c main.c
kbd.o : kbd.c defs.h command.h
		cc -c kbd.c
command.o : command.c defs.h command.h
		cc -c command.c
display.o : display.c defs.h buffer.h
		cc -c display.c
insert.o : insert.c defs.h buffer.h
		cc -c insert.c
search.o : search.c defs.h buffer.h
		cc -c search.c
files.o : files.c defs.h buffer.h command.h
		cc -c files.c
utils.o : utils.c defs.h
		cc -c utils.c
		
clean :
		rm edit $(objects)
```

## make 推断 recipes

隐含的规则：默认使用 “cc -c” 命令把 .c 文件编译成 “.o” 文件，所以如果使用该命令，则可以省略。另外，由于.o文件默认会寻找名字相同的 .c 文件作为依赖，所以在 prerequisites 中可以省略名字相同的 .c 文件。

```
objects = main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
		
edit : $(objects)
		cc -o edit $(objects)
		
main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
		rm edit $(objects)
```

## 另一种写法

```
objects = main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
		
edit : $(objects)
		cc -o edit $(objects)
		
$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h
```

