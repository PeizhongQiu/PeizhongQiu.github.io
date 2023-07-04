---
title: __builtin_系列函数
tags: C++
abbrlink: 7e0b4172
date: 2021-03-21 00:00:00
---

**\_\_builtin\_ffs (unsigned int x)**
返回x的最后一位1的是从后向前第几位，比如8（1000）返回4。

<!-- more -->

**\_\_builtin\_clz (unsigned int x)**
返回前导的0的个数。
**\_\_builtin\_ctz (unsigned int x)**
返回后面的0个个数，和\_\_builtin\_clz相对。
**\_\_builtin\_popcount (unsigned int x)**
返回二进制表示中1的个数。
**\_\_builtin\_parity (unsigned int x)**
返回x的奇偶校验位，也就是x的1的个数模2的结果。

这些函数都有相应的usigned long和usigned long long版本，只需要在函数名后面加上l或ll就可以了，比如 \_\_builtin\_clzll。

版权声明：本文为CSDN博主「Yuer-」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/yuer158462008/article/details/46383635