---
layout: post
comments: true
title: The Direct-to-Data (D2D) Cache:Navigating the Cache Hierarchy with a Single Lookup 
show: 0
---

>D2D在TLB中扩展了cache line的位置信息，其中包括了cache line所属的cache层次和cache line的way信息

* 想一想传统cache的访问流程，先从高级cache开始访问，没有找到再逐级往下层探测->延迟与能耗
* 不仅把cache line所在的cache层次信息记录下来了，甚至把cache所在的way的信息也保存下来了（那么根据cache替换算法，需要发生promotion的时候，TLB就需要更新？）
	* 那么访问时直接找到所在cache层次（如果在cache中的话），而且省去了tag区的查找，直接访问data区。不在cache中的情况直接bypass去访问主存。
* exclusive和inclusive架构信息记录的方式应该有所不同（**D2D采用exclusve架构**）
* **一个问题是为什么需要HUB，首先理解HUB的意义是什么？eTLB(extended TLB)和HUB到底存储着什么信息？这些信息谁的？（能不能把HUB的信息改成存放在L2 cache中）**
* eTLB entry中有一个CLT valid bit，如果这个标志位处于置位状态，有效的cache line位置信息存储在eTLB中，否则在HUB中（相应的，HUB entry中存在一个eP）
* 所有的cache line必须属于HUB中的一个有效页
* 必须要思考的一个问题是采用eTLB辅助的方式比原先传统cache的访问方式，在L1 cache命中时是否存在优势？
![](https://github.com/dioxygen/markdown/raw/master/image/D2Dpaper/D2D框架.png)

> 访问cache发生不同情况下的流程

1. eTLB hit
![1](https://github.com/dioxygen/markdown/raw/master/image/D2Dpaper/eTLBhit.png)
	* eTLB hit + L1 hit (94%访存)
		* 流程见fig.2(a)，访问eTLB中CLT的3个bit就可以直接读取L1 cache中的正确way（这里隐含了一个条件，**L1 cache的set数与cache block大小乘积与page大小的关系**）
	* eTLB hit + L2 hit (3%访存)
		* 需要读取虚拟地址的最低有效位的一部分，还需要从eTLB中获得一部分物理地址（**4 bits**对1MB 16way L2 Cache来说），然后需要从eLB的CLT中获取way index信息
		* 流程见fig.2(b)，在从L2获取数据后，需要进入替换，在将L1中不命中L2中命中的数据发送到L2之前需要先将L1中的一个cache line踢出去，然后将L1cache不命中的数据从L2移动到L1（在此期间需要更新HP和CLT）
		* 通过访问eTLB可以直接读取L2，而不需要采用逐级试探的方法
	* eTLB hit + Cache miss (2%访存)
		* 和eTLB hit + L2 hit流程类似，具体见fig2.(c)
		* eTLB中的信息使得直接访问DRAM，从而绕过了整个cache层次结构
2. eTLB miss
		当整个eTLB page缺失或者CLT的valid bit没有置位的时候会发生eTLB缺失，因此需要去Hub中寻找cache line的位置信息
![](https://github.com/dioxygen/markdown/raw/master/image/D2Dpaper/eTLBmiss.png)
	* eTLB miss + Hub hit（0.7%访存）
		当eTLB缺失时也需要进行和L1 cache缺失时类似的操作，即需要选择一个eTLB victim进行淘汰（需要将active的CLT数据写回到Hub中），然后将不命中的信息新装入eTLB中（需要无效化eP指针，指示该页不在eTLB中，后面新填入的eTLB信息需要新增eP指针）
		* 流程见fig3(d)，当eTLB缺失时需要去Hub中寻找cache line的位置信息，然而Hub是以物理地址方式访问（虚实地址界面见fig.1）的，因此需要先访问L2 TLB进行虚拟地址到物理地址的转换
		* **总的来讲由于eTLB除了存储传统的虚实地址转换信息外还保存着cache line的位置信息，因此需要从两个地址调取信息，并且第二个地方（Hub）的访问依赖于前一个地方的访问（L2 TLB），同时写回和踢出也是在eTLB和Hub之间进行的（尝试理解一下eP和HP的意义？）**
	* eTLB miss + Hub miss （0.4%访存）
		* 由于Hub未名中，因此又要发生上面其他部件不命中时类似的操作，首先需要选择victim，但是要注意的是：为了维持一致性，所有属于踢出Hub项的page的cache line必须也都要被踢出（**不好维持崩溃一致性？**）
		*流程见fig3(e)， 第四步中新页装入Hub中时为什么需要将CLT信息设置为Not Cached？
3. Cache line eviction（4.5%访存，不在关键路径上）
	* 当L1cache中的cache line需要淘汰到L2中时，由于D2D采用CLT方式记录cache line的位置信息，因此当cache line的位置发生移动时，需要更改CLT信息
	* 流程见fig3(f)，关键是理解其中第二步HP为什么能够用来寻找L1中cache line在L2中存放的位置？HP是用来寻找CLT项并更新，eP用来确定active状态的CLT是在eTLB中还是在Hub中
4. 兼容性
	*需要额外考虑：synonyms, JITed/dynamically generated code, large pages, atomicity, and coherency
5. 改进D2D使之变得更加高效
	*改进cache布局，最小化由于Hub替换导致的cache line踢出数量
6. 测试
	* 实验方法与环境设置
		* 采用gem5模拟器
		* 对照组的设置：传统具有tag的cache和具有更快L2访问时间的cache（通过并行访问tag与data区减少延迟，但是增加了能耗）
		* 主要测试了减小L1 cache大小的情况不同方案的加速比以及能耗变化，在测试结果中D2D方案在性能与能耗上均优于其他两种
		* 同时由于D2D的L2 cache访问延迟减小，因此对ROB的压力也越小，因此除了考虑减少L1 cache大小之外也可以考虑减少ROB表项但不影响系统性能
扩展了TLB的表项内容，通过一种粗粒度的方式记录了cache line的位置信息
