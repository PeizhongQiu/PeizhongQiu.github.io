---
title: 资源调度算法调研（一）
tags: 资源调度
abbrlink: bcc3e70e
date: 2021-10-26 09:36:04
---


## 传统算法<sup>[1]</sup>

<!-- more -->

### 概念

<font color=#FF0000>metatask</font>: a collection of independent tasks with no intertask data dependencies;

<font color=#FF0000>&tau;</font>: The size of the metatask (i.e., the number of tasks to execute);

<font color=#FF0000>&mu;</font>: the number of machines in the HC suite;

&tau; and &mu; are static and known beforehand.

It is assumed that an accurate estimate of the expected execution time for each task on each machine is known prior to execution and contained within a &tau;*&mu; ETC (expected time to compute) matrix;

It is also assumed that each machine executes a single task at a time.

<font color=#FF0000>Machine availability time, mat(m<sub>j</sub>)</font>：the earliest time machine m<sub>j</sub> can complete the execution of all the tasks that have previously been assigned to it (based on the ETC entries for those tasks). 

<font color=#FF0000>The completion time for a new task t<sub>i</sub> on machine m<sub>j</sub> , ct(t<sub>i</sub> , m<sub>j</sub>)</font>: the machine availability time for m<sub>j</sub> plus the execution time of task t<sub>i</sub> on machine m<sub>j</sub> , i.e., ct(t<sub>i</sub> , m<sub>j</sub>)= mat(m<sub>j</sub>)+ETC(t<sub>i</sub> , m<sub>j</sub>). 

The performance criterion used to compare the results of the heuristics is the maximum value of ct(t<sub>i</sub> , m<sub>j</sub>), for  0 &le; i < &tau; and 0 &le; j < &mu;. The maximum ct(t<sub>i</sub> , m<sub>j</sub>) value, over 0 &le; i < &tau; and 0 &le; j < &mu;, is the metatask execution time, and is called the <font color=#FF0000>makespan</font> . 

Each heuristic is attempting to minimize the makespan, i.e., finish execution of the metatask as soon as possible.

### 算法

| 算法 | <p align="middle">描述</p> | <p align="middle">优点</p> | <p align="middle">缺点</p> |
|  :--:  | :---  |  :---  |  :---  |
| Opportunistic Load Balancing (OLB) | OLB算法会将负载随机交给一个可用的节点，不管任务执行的时间。动机是尽量保持所有的机器忙碌。 | <p align="middle">简单</p> | poor makespan |
| Minimum Execution Time (MET) | MET会将任务分配给对于该任务，有着最小执行时间的机器，不管机器何时可用。动机是尽量将任务分配给最适合它的机器。 |  | 会造成严重的负载不均衡 |
| Minimum Completion Time (MCT) | MCT会将任务分配给对于该任务，有着最小完成时间的机器。 |  |  |
| Min-min | U：unmapped tasks；M={min<sub>0&le;j<&mu;</sub>(ct(t<sub>i</sub> , m<sub>j</sub>)), for each t<sub>i</sub> &in;U}；M中最小的任务分配给对应的机器，之后将该任务移除，重复该步骤直到U为空。 | good makespan | 执行长任务而带来延迟。比如：当有很多短任务和一个很长的任务，会由于最后执行长任务而带来极大的开销 |
| Max-min | U：unmapped tasks；M={min<sub>0&le;j<&mu;</sub>(ct(t<sub>i</sub> , m<sub>j</sub>)), for each t<sub>i</sub> &in;U}；<br>M中最大的任务分配给对应的机器，之后将该任务移除，重复该步骤直到U为空。 | 最小化由于执行长任务而带来的延迟。比如：当有很多短任务和一个很长的任务，Min-Min会由于最后执行长任务而带来极大的开销，这时Max-Min算法会更优。 | 其他时候调度策略都不够好。短任务的等待时间过长。 |
| Duplex | 会同时执行Min-Min和Max-Min，然后选择更好的一个 |  |  |
| Round-Robin(RR) | Round-Robin 算法将负载均匀的发放给各个节点。当使用这个算法时，调度器会循环的将VM分配给节点。比如：3个节点，第一个请求到达时，分配给第一个节点，第二个请求到达时，分配给第二个节点，第三个请求到达时分配给第三个节点，第四个请求到达时又分配给第一个节点，以此类推。 | 能均匀的利用所有的资源，保障了公平性 | 没有考虑任务的执行时间和资源使用情况。 |
| Power Save Algorithm | 在调度器分配了VM后，检查是否有空闲的节点，有的话就关掉。等到有请求需要分配到该节点时，再打开该节点。 | 考虑能耗问题 |  |
| Genetic Algorithms (GA) | initial population genetation;<br>evaluation;<br>while(stopping criteria not met){<br>&emsp;selection;<br>&emsp;crossover;<br>&emsp;mutation;<br>&emsp;evaluation;<br>}<br>output best solution;<br>具体算法可以参考 | Genetic Algorithms (GAs) are a technique used for searching large solution spaces.  GA usually found the best mappings of all 11 heuristics. | 调度时间长 |
| Simulated Annealing (SA) | 具体算法可以参考 |  | 可能会找到比Min-Min和GA更差的解决办法；调度时间较长 |
| Genetic Simulated Annealing (GSA) | a combination of the GA and SA techniques . In general, GSA follows procedures similar to the GA outlined above. However, for the selection process, GSA uses the SA cooling schedule and system temperature and a simplified SA decision process for accepting or rejecting a new chromosome. |  | 调度时间长 |
| Tabu | {% asset_img  tabu.PNG 图1 tabu%}<br>具体算法可以参考 |  | 调度时间较长 |
| A* | 具体算法可以参考 |  | 调度时间很长 |

### 总体评价：

- 基本没有考虑能耗情况，所有的节点都是一开始就准备好，一直开启；
- 假设很多也已经不符合现在的情况，比如：现在一个节点都是多线程任务，很少一个机器一个时间只有一个任务执行；
- 很少考虑节点是否有充足的资源，这可能是因为一个时间一个机器只有一个任务，也就是说，在一个时间，机器所有的资源都会给到该任务，所以不考虑，但是，在多进程的环境下，资源问题也需要考虑；
- 不支持热迁移
- 这些算法都是静态的算法，有些还是批处理的，提前会知道很多信息。



## 论文算法

### AN ADAPTIVE ALGORITHM FOR DYNAMIC PRIORITY BASED VIRTUAL MACHINE SCHEDULING IN CLOUD<sup>[2]</sup>

#### 伪代码

```
Input: None
Output: None
Algorithm sched_priority
{ 
	If(P1 is not set)
		P1=max available resource node
		If(P1 is turned OFF)
			Turn P1 ON
	If(load factor of P1<0.8)
		Assign VM to P1
	Else if(P2 is set AND load factor of P2<0.8)
		Swap P1 and P2;
		Assign VM to P1;
	Else
		P2=P1
		P1=current max available resource node 
 
		If(P1 is turned OFF)
			Turn P1 ON
		Assign VM to P1;
	Turn OFF all unused nodes;
}
```

#### 说明

在初始状态下，P1和P2都是没有设置的。当一个请求到达时，会判断是否设置了P1，如果没有设置P1，则将P1设置为拥有最多空闲资源的节点，如果此时该节点没有开启，则将该节点开启；之后判断P1的负载因子是否小于0.8，如果小于0.8，说明P1还能够放其他资源，就将资源放入P1对应的节点中；如果P1的负载因子大于0.8，P2被设置了且负载因子小于0.8，则交换P1和P2，将资源交给P1对应的节点；如果P2没有被设置，或者负载因子也大于0.8，则将P2设置为P1的节点，并将P1设置为当前拥有空闲资源最多的节点。最后，关掉那些没有使用的空闲节点。

#### 评析

优点：考虑了能源问题，在有空闲节点时，能够关闭掉那些空闲的节点；可拓展性较好；给了伪代码；
缺点：实验简单，虚拟机的数量过少，只有10个虚拟机，实验过程省略了；缺少热迁移的考虑，即缺少任务整合的部分，那么，是否可以在此基础上增加一个任务整合的模块？即当某个主机的负载低于某个阈值，比如10%时，将主机的内容迁移到其他主机上，并将该主机关闭；资源分很多种，比如：内存，vcpu等等，不同类型一次申请的量可能不同，可能在某一负载下只能满足其中部分，不能满足全部；。

### Dynamic Round-Robin (DRR)<sup>[3]</sup>

DRR是在传统RR的基础上，增加了两条规则：第一，在一个虚拟机结束时，该主机将不会接受新的虚拟机，这种状态称为“退休状态”，等该主机的所有VM都结束，关闭该主机；第二，如果主机在“退休状态”呆了足够长的时间，就将主机的所有VM热迁移到其他主机，然后将主机关闭。

#### 评析

优点：考虑了能耗；支持live migration；
缺点：没有源代码；实验部分阐述过于简单，没有说明实验的环境，数据量大小，甚至过程都省略了；没有参考文献；如果同时有很多主机的虚拟机都同时结束了，那么这些主机都不能接受新的虚拟机，那么当新的虚拟机到来时，就只能轮询少量的虚拟机，这可能会使少部分的虚拟机过度使用；传统Round-Robin的问题也会发生在这里，比如：故障处理；如果虚拟机退出时，该主机有很多的虚拟机，迁移的开销就会很高。

### Power Aware Load Balancing for Cloud Computing（PALB）<sup>[4]</sup>

#### 伪代码

````
balance:
	for all active compute nodes j ∈ [m] do
		n[j] = current utilization of compute node j
	end for
	if all n[j] > 75% utilization //all available nodes are active
		boot vm on most underutilized n[j]
	end if
	else
		boot vm on most utilized n[j], which can also fulfill requirements
	end else
upscale:
	if each n[j] > 75% utilization
		if n[j] < m
			boot compute node n[j+1]
		end if
	end if
downscale:
	if vmi idle > 6 hours or user initiated shutdown
		shutdown vmi
	end if
	if n[j] has no active vm
		shutdown n[j]
	end if
````



#### 说明

该算法的目标是在维持节点可用性的同时，降低能耗。该算法主要有三个过程：balance，upscale，downscale。

- balance过程确定虚拟机应该安装的位置。首先获取当前所有活跃节点的利用率，当所有结点的利用率都大于75%时，将虚拟机分配到利用率最低的节点，否则将虚拟机分配到利用率最高的，且可以满足需求的节点。
- upscale过程是当所有虚拟机节点的利用率都大于75%时，创建一个新的虚拟机
- downscale过程是当虚拟机长时间闲置时，关闭虚拟机，而当一个主机没有活跃的虚拟机时，关闭主机。

#### 评析

优点：实验做得比较详细，说明了实验环境和步骤；考虑了能源因素，能够将长时间未使用的虚拟机关掉，将未使用的主机关掉；资源利用率高；考虑了异质的问题
缺点：为考虑热迁移，即如果一个主机的负载过低，将主机的虚拟机迁移，之后关闭虚拟机；在虚拟机运行的时候，并不是满负载的，对于申请时要求的资源小于实际使用的资源的情况缺少考虑；实验部分虚拟机个数有点少，只测试了20个和30个虚拟机。

### Load Balanced Min-Min Algorithm(LBMM)<sup>[5]</sup>

#### 伪代码

**CT<sub>i</sub><sub>j</sub>**: completion time of task i on machines j 

**ET<sub>i</sub><sub>j</sub>**: expected execution time of job i on resource j  

**Rj**: ready time or availability time of resource j after  completing the previously assigned jobs.

````
for all tasks Ti 
	for all resources 
		Cij=Eij+rj 
	do until all tasks are mapped 
 		for each task find the earliest completion time and the resource that obtains it 
		find the task Tk with the minimum earliest completion time 
		assign task Tk to the resource Rl that gives the earliest completion time 
		delete task Tk from list
		update ready time of resource Rl
		update Cil for all i 
	end do 
// rescheduling to balance the load
sort the resources in the order of completion time
for all resources R
	Compute makespan = max(CT(R))
End for
for all resources
	for all tasks
		find the task Ti that has minimum ET in Rj
		find the MCT of task Ti
		if MCT < makespan
			Reschedule the task Ti to the resource that produces it 
			Update the ready time of both resources
		End if
	End for
End for
//Where MCT represents Maximum Completion Time
````

#### 说明

这个算法是为了解决在Min-Min算法长任务滞后带来的问题。该算法主要分成两个部分，第一部分使用传统的Min-Min算法得到一般的解决方案；第二部分对第一部分的解决方案进行重新调度，首先将所有的节点根据完成时间进行排序，然后对每个节点中的每个任务进行调度，如果再次调度后可以得到更小的最大完成时间，则将任务进行调度，并更新信息。

#### 评析

优点：在原来Min-Min的基础上增加了重新调度的过程，降低了makespan
缺点：没有摆脱传统算法的限制，直接使用传统算法的假设，因此，传统算法的缺点这个方法仍然有；实验过于简单，最后资源的实验说的很含糊



## 参考文献

[1] Braun T D ,  Siegel H J ,  Beck N , et al. A Comparison of Eleven Static Heuristics for Mapping a Class of Independent Tasks onto Heterogeneous Distributed Computing Systems[J]. Journal of Parallel & Distributed Computing, 2001, 61(6):810-837.

[2] Subramanian S ,  Krishna N G ,  Kumar K M , et al. An Adaptive Algorithm for Dynamic Priority Based Virtual Machine Scheduling in Cloud[J]. International Journal of Computer Science Issues, 2012, 9(6).

[3] Lin C C ,  Liu P ,  Wu J J . Energy-Aware Virtual Machine Dynamic Provision and Scheduling for Cloud Computing[J]. IEEE, 2011.

[4] Galloway J M ,  Smith K L ,  Vrbsky S S . Power Aware Load Balancing for Cloud Computing[J]. lecture notes in engineering & computer science, 2011.

[5] Kokilavani T ,  Amalarethinam D G . Load Balanced MinMin Algorithm for Static MetaTask Scheduling in Grid Computing[J]. International Journal of Computer Applications, 2011, 20(2):42-48.

