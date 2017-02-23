---
title: 扒了扒libdispatch源码
date: 2017-02-21 15:47:10

categories:
- Objective-C

tags:
- libdispatch
- GCD
---

前几天扒了一下libdispatch的源码，当前最新的是 libdispatch-703.30.5.tar，之前也在网上查了一些博客，发现源码有点对不上，应该是代码比较老了吧。之前对GCD的认识都比较浅：

* 一直有个误区认为 ***dispatch_sync*** 是卡住当前线程，然后去异步线程执行，原来并不是如此；
* 一直认为 ***dispatch\_sync*** 死锁的是线程，原来并不是如此；
* 一直不是很清楚 ***libdispatch*** 中 ***队列*** 和 ***线程*** 是如何协作的
* 一直想知道 ***dispatch_barrier*** 是怎么实现的
* ......

好了，来来来，扒扒源码就全知道了，当然源码中我还是有很多疑惑的地方，希望大家拍，反正我只是搬运工，哈哈😄
***

# 全景 #

### 结构图 ###
先看这张泛滥了的图吧：

![gcd-pool](/images/gcd-pool.png)

从上图我们可以得到的信息是：

1. 只有 ***Main-Queue*** 的任务可以提交到 ***Main-Thread*** 上
2. 系统提供的所有的 ***Default-Queues*** 会共享一个 ***Thread-Pool***
3. 我们自己创建的所有 ***Custom-Queues*** 都会和 ***Default-Queues*** 有某种联系
4. 关于线程的任务调度用户不能也不需要参与

### 关键数据结构 ###
先记下几个关键的数据结构吧，等会儿好返回来查，有个东西提前说明一下：

这些结构体都会有两个，一个是 ***\*\*\*_t*** 另一个是 ***\*\*\*_s*** ，其中 ***\*\*\*_t*** 是 ***\*\*\*_s*** 的指针类型，_s是结构体。比如 dispatch\_queue\_t 和 dispatch\_queue\_s。

***dispatch\_object\_s*** 这个结构体可以代表所有的 ***gcd对象***，我说OC中的 ***id*** 类型，你一定就知道是什么了。

#### dispatch\_continuation\_s ####
``` c
struct {
	struct dispatch_object_s *volatile do_next; // 下一个任务
	dispatch_function_t dc_func;                // 执行的方法
	void *dc_ctxt;                              // 方法上下文
	void *dc_data;                              // 相关数据
	void *dc_other                              // 其它信息
}
```

#### dispatch\_queue\_s ####
``` c
struct {
	struct dispatch_queue_s *do_targetq;               // 目标队列，这个最终会指向一个系统的默认队列
	struct dispatch_object_s *volatile dq_items_head;  // 队列头部
	struct dispatch_object_s *volatile dq_items_tail;  // 队列尾部
	unsigned long dq_serialnum;                        // 队列序号
	const char *dq_label;                              // 队列名
	dispatch_priority_t dq_priority;                   // 优先级
	dispatch_priority_t volatile dq_override;          // 是否被覆盖
	uint16_t dq_width;                                 // 可并发执行的任务数
	dispatch_queue_t dq_specific_q;                    // 特殊队列
	uint32_t dq_side_suspend_cnt;                      // 暂停的任务数
}
```

---
未完，明天继续...