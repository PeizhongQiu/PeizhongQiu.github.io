---
title: cpu利用率
tags: 编程之美读书笔记
abbrlink: 357e56ad
date: 2021-11-26 14:40:28
---

## 目标

写一个程序，让用户来决定 Windows 任务管理器的CPU占用率。程序越精简越好，语言不限。比如：

- CPU占用率固定在50%，为一条直线；

- CPU占用率为一条直线，但是具体占用率由用户决定（参数范围1-100）；

- CPU占用率状态是正弦曲线。

  <!-- more -->

## 思路

在任务管理器的一个刷新周期里，CPU忙（执行应用程序）的时间和刷新周期总时间比率就是CPU的占用率，也就是说，任务管理器中显示的是每个刷新周期内CPU占用率的统计平均值。因此，可以写一个程序，让他在任务管理器的刷新期间内一会儿忙，一会儿闲，然后通过调节忙/闲的比例，就可以控制任务管理器中显示的CPU占用率。

### 解法1

由于现在使用的windows电脑一般是多核，为了能够让程序能够在单个cpu上运行，可以在<font color=#FF0000>任务管理器</font>中<font color=#FF0000>详细信息</font>栏选中运行的程序，右键点击，选择<font color=#FF0000>设置相关性</font>，并在跳出的界面里选择程序运行所在的cpu。

```C
#include <stdio.h>

int main(){
	for(;;);
	return 0;
}  
```

首先查看死循环的CPU资源利用率。从图中可以发现，cpu的利用率达到了100%。

{% asset_img  cpu1.PNG 图1 cpu1%}

对上面的代码进行修改。增加sleep函数。现在我使用的电脑的cpu主频是1.60GHz，即每秒1.6\*10<sup>9</sup>时钟周期。假设每个时钟周期运行两条代码。循环转换成汇编大约5条指令。于是，cpu一秒就可以运行(1.6\*10<sup>9</sup>\*2) / 5 = 3.2\*10<sup>8</sup>个循环。于是，我们写了第二版代码。

```C
#include <stdio.h>

int main(){
	for(;;){
		int i;
		for(i = 0; i < 320000000; i++);
		Sleep(1000);
	}
	return 0;
}  
```

运行结果下图所示。我们可以看到呈现为锯齿状，这是因为CPU工作一秒，休息一秒。而我们的刷新周期是小于一秒的，所以会呈现这样的曲线。

{% asset_img  cpu2.PNG 图2 cpu2%}

显然，这不符合我们的要求，我们尝试增加循环的频率，将320000000和1000降低两个数量级。注意，sleep时间不能太小，太小会造成线程频繁挂起和唤醒，无形中增加了内核时间的不确定性因素。

```C
#include <stdio.h>

int main(){
	for(;;){
		int i;
		for(i = 0; i < 3200000; i++);
		Sleep(10);
	}
	return 0;
}  
```

从下图中可以看出cpu占用率在60左右浮动。通过不断调整3200000参数，就能在一条指定的机器上获得一条大致稳定的50%CPU占用率曲线。

{% asset_img  cpu3.PNG 图3 cpu3%}

经过一番调试之后，最终参数大约为2400000，最终效果图如下图所示。

{% asset_img  cpu4.PNG 图3 cpu4%}

使用这种方法，有两点注意：

- 尽量减少sleep/awake频率，减少操作系统内核调度程序的干扰。

- 尽量不要调用system call，因为这也会导致很多不可控的内核运行时间。

该方法有个很严重的问题：不能适应机器差异性。一旦换了CPU，必须要重新评估。

### 解法2

GetTickCount()可以获得系统启动到现在所经历时间的毫秒值，最后能够统计到49.7天。

````C
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

int main(){
	int busyTime = 10;
	int idleTime = busyTime;
	
	long long startTime = 0;
	for(;;){
		startTime = GetTickCount();
		while((GetTickCount() - startTime) <= busyTime);
		Sleep(idleTime);
	}
	return 0;
}  
````

注意这两种方法都是假设当前cpu只有当前程序运行，但实际上操作系统中有很多程序会同时执行各种各样的任务，因此，实际运行的效果会如下图所示。

{% asset_img  cpu5.PNG 图5 cpu5%}

那么，我们该怎样做呢？这就要运用到另一个工具perfmon.exe。

Perfmon可以获得有关操作系统，应用程序和硬件的各种能效计数器。