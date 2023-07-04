---
title: 'Dash: Scalable Hashing on Persistent Memory'
tags: NVM论文研读
abbrlink: 6ab0c77
date: 2021-03-21 05:27:22
---
## dash bucket结构
Dash的总体结构如图1所示。从图中可以看出，该结构和CCEH同样采取了可拓展的哈希结构，并且同样有三个层次，一个是Directory层，一个是Segment层，一个是Bucket层。不同的是，每个Segment层除了正常的Bucket外，还有Stash buckets用来存储冲突的键值对。根据论文，一个Segment层的Stash buckets数量为2或4。另外，在Directory层有三个主要变量，Lock是锁，Version用来做版本控制，Clean用来判断系统的关闭是否是干净的。

<!-- more -->

{% asset_img  dash_architecture.PNG 图1 Dash总体结构%}

图2是bucket层的具体结构。从图中可知，一个bucket就只存储14个键值对，一个键值对16个字节。另外，每个bucket有32字节的元数据，其中包括4字节的版本锁，根据代码可知，最高位是互斥锁，后面31位作为版本号进行版本控制；接着的counter是用来计算bucket中有多少个键值对；membership是一个位图，用来显示那些不是直接索引到该bucket中的键值对；Alloc.bitmap也是一个位图，用来显示该bucket中的全部键值对；接下来是18个fingeprint，每个fingerprint是1字节，所谓的fingerprint是哈希值的后8位，通过fingerprint可以大量减少NVM读的次数，有一点值得注意，一个bucket只有14个键值对，但是却有18个fingerprint，这是stash bucket中在本bucket溢出的键值对的fingerprint；overflow fingerprint bitmap是4位，判断溢出的对应的fingerprint是否被占用；overflow bit用来判断这个bucket是否有溢出的键值对，stash bucket index用来判断fingerprint对应的键值对在哪个bucket；overflow membership用来判断溢出的键值对是否直接索引到该bucket ；overflow count表示直接对应该bucket的键值对却无法插入到该bucket和下一个bucket的键值对数量，注意overcount不包括那些在bucket或下一个bucket有存储fingerprint的键值对。

{% asset_img  dash_bucket.PNG 图2 bucket结构%}

## dash实现细节
每个Segment有64个bucket，因此可以根据6位来确定bucket的位置。哈希值最后8位是作为fingerprint，最后9到14位来确定bucket位置。

### dash insert
最上层的伪代码如图3所示。首先，是判断键值是否已经存在，如果存在，则直接返回。其次，如果target\_bucket和probing\_bucket都未满，为了让两者负载均衡，会插入到两者数量较少的一边；如果两者都满了，则会判断target和probing target中是否有可以移动的键值对，即在target中属于前一个bucket的键值对，且前一个bucket有空余的位置，则将其中一个插入到前一个bucket，或者probe target中属于probe target的键值对，且后一个bucket有空余的位置，则将其中一个插入到前一个bucket。最后，如果没有空闲的位置，则插入到stash中，并设置target的overflow为1，如果stash中也没有空余的位置，则分裂。接下来，对其中的有些细节进行补充。

{% asset_img  insert_top.PNG 图3 dash insert伪代码%}

### key exists
根据源代码，首先是在target\_bucket和probing\_bucket中寻找是否存在相同的键值，这个部分可以利用fingerprint大量减少读的次数，并且，fingerprint可以使用SSE加速，另外，在源代码中，对14位每位进行判断的时候，使用了循环展开的方法降低开销；之后，判断是否需要去stash中查找，判断是否要在stash中查找要依次查看overflow bit是否为0，overcount是否为0，overflowBit与overflow fingerprint与overflow membership的关系，不满足则要对stash进行搜索。

### bucket insert
首先找到空闲的键值对，这可以利用bitmap进行位运算得到。接着插入，并设置标志位。由于clflush一次刷新一个cacheline，一个cacheline为64字节，所以所有的元数据都可以在一次刷新中刷入。

### split
Split过程基本与CCEH相似。重点说明几点不同之处：第一，分裂的时候，dash要将所有bucket的锁都上锁，第二，dash在分裂时候的正确性由libpmemobj的事务保障，这保证分裂出去的Segment在程序崩溃之后能够回收成功。

### search

search伪代码如图4所示。从伪代码中，可以看出，在搜索的过程中并没有使用锁。dash采用乐观锁的方式大大提高并行环境下的搜索效率。在搜索之前，判断bucket是否上锁；在搜索完成后，dash通过检查bucket的版本是否与之前读取的一致来判断search的过程中是否有进程对bucket进行修改，如果有，则会重新读取。另外，由于fingerprint的存在，大大减少了读的次数。

{% asset_img  dash_search.PNG 图4 dash search伪代码%}

## dash 与 cceh评测

### cceh改写
dash的作者对cceh进行了改写，首先在insert上，作者增加了判断新增的键值对是否存在的判断，由于cceh没有此方面的实现，所以作者相当于搜索了一次；另外，在split的过程中，作者将cceh的分裂用libpmemobj的事务进行处理。

### 单线程比较

对于key大小大于8字节的键值对来说，由于每次读取键值都需要先读指针，再读key，这就体现了fingerprint的优越性，fingerprint大大减少了读的次数。

{% asset_img  re1.PNG 图5 单线程%}

### 多线程比较

{% asset_img  re2.PNG 图6 多线程%}

### 负载因子

{% asset_img  re3.PNG 图7 负载因子%}

## dash代码运行环境要求
源代码地址 https://github.com/baotonglu/dash.git
linux 内核版本4.17之后
glibc 版本2.29之后