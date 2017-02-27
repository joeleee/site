---
title: 扒了扒libdispatch源码
date: 2017-02-21 15:47:10

categories:
- Objective-C

tags:
- libdispatch
- GCD
---

前几天扒了一下libdispatch的源码，当前最新的是 libdispatch-703.30.5，之前也在网上查了一些博客，发现源码有点对不上，应该是代码比较老了吧。之前对GCD的认识都比较浅：

* 一直有个误区认为 ***dispatch_sync*** 是卡住当前线程，然后去异步线程执行，原来并不是如此；
* 一直认为 ***dispatch\_sync*** 死锁的是线程，原来并不是如此；
* 一直不是很清楚 ***libdispatch*** 中 ***队列*** 和 ***线程*** 是如何协作的
* 一直想知道 ***dispatch_barrier*** 是怎么实现的
* ......

好了，来来来，扒扒源码就全知道了，当然源码中我还是有很多疑惑的地方，希望大家拍，反正我只是搬运工，哈哈😄
***

# 全景结构图 #
先看这张泛滥了的图吧：

![gcd-pool](/images/gcd-pool.png)

从上图我们可以得到的信息是：

1. 只有 ***Main-Queue*** 的任务可以提交到 ***Main-Thread*** 上
2. 系统提供的所有的 ***Default-Queues*** 会共享一个 ***Thread-Pool***
3. 我们自己创建的所有 ***Custom-Queues*** 都会和 ***Default-Queues*** 有某种联系
4. 关于线程的任务调度用户不能也不需要参与
---

# 关键数据结构 #
先记下几个关键的数据结构吧，等会儿好返回来查，有个东西提前说明一下：

这些结构体都会有两个，一个是 ***\*\*\*_t*** 另一个是 ***\*\*\*_s*** ，其中 ***\*\*\*_t*** 是 ***\*\*\*_s*** 的指针类型，_s是结构体。比如 dispatch\_queue\_t 和 dispatch\_queue\_s。

***dispatch\_object\_s*** 这个结构体可以代表所有的 ***gcd对象***，我说OC中的 ***id*** 类型，你一定就知道是什么了。

## dispatch\_continuation\_s ##
``` c
struct dispatch_continuation_s {
	struct dispatch_object_s *volatile do_next; // 下一个任务
	dispatch_function_t dc_func;                // 执行的方法
	void *dc_ctxt;                              // 方法上下文
	void *dc_data;                              // 相关数据
	void *dc_other                              // 其它信息
}
```
dispatch\_continuation\_s 是中的任务的结构体，被传入的 ***block*** 会被变成这个结构体对象塞入队列

## dispatch\_queue\_s ##
``` c
struct dispatch_queue_s {
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
	const struct queue_vtable_s *do_vtable {           // 队列的一些函数指针
		unsigned long const do_type;               // 队列类型，例如：DISPATCH_QUEUE_CONCURRENT_TYPE、DISPATCH_QUEUE_SERIAL_TYPE、DISPATCH_QUEUE_GLOBAL_ROOT_TYPE ...
		const char *const do_kind;                 // 队列种类，例如："serial-queue"、"concurrent-queue"、"global-queue"、"main-queue"、"runloop-queue""mgr-queue" ...
		void (*const do_dispose)(/*params*/);      // 销毁队列
		void (*const do_suspend)(/*params*/);      // 暂停队列
		void (*const do_resume)(/*params*/);       // 恢复队列
		void (*const do_invoke)(/*params*/);       // 开始处理队列
		void (*const do_wakeup)(/*params*/);       // 唤醒队列
		void (*const do_set_targetq)(/*params*/);  // 设置target queue
	};
}
```
dispatch\_queue\_s是队列的结构体，在它的 ***do_vtable*** 中有很多函数指针，对应队列的一些操作方法，对应有一些宏可以调用队列中的这些方法。比如，*** do_dispose*** 方法对应有一个宏 ***dx_dispose*** ：

``` c
#define dx_dispose(queue) &(queue)->do_vtable->_os_obj_vtable->do_dispose(queue)
```

关键就这两个数据结构 ***dispatch\_continuation\_s*** 和 ***dispatch\_queue\_s***。

---


# dispatch系列常用的方法 #
接下来我们看看dispatch系列几个常用的方法吧。

## dispatch\_queue\_create ##
``` c
// skip zero
// 1 - main_q
// 2 - mgr_q
// 3 - mgr_root_q
// 4,5,6,7,8,9,10,11,12,13,14,15 - global queues

unsigned long volatile _dispatch_queue_serial_numbers = 16;

dispatch_queue_t dispatch_queue_create(const char *label, dispatch_queue_attr_t dqa) {
    if (DISPATCH_QUEUE_SERIAL == dqa) {
        dqa = _dispatch_get_default_queue_attr();
    }

    // 首先获取一个全局队列队列tq，第一个参数是优先级，第二个参数是是否允许过量
    dispatch_queue_t tq = _dispatch_get_root_queue(dqa->dqa_qos_class, dqa->dqa_overcommit);

    // 创建并初始化队列dq，串型队列宽度为1，并行队列宽度为最大值
    dispatch_queue_t dq = _dispatch_alloc(dispatch_queue, sizeof(struct dispatch_queue_s) - DISPATCH_QUEUE_CACHELINE_PAD);

    // 初始化队列，队列的 dq_serialnum 会对 _dispatch_queue_serial_numbers 进行自增
    _dispatch_queue_init(dq, dqf, dqa->dqa_concurrent ? 32766 : 1, dqa->dqa_inactive);

    //:设置do_targetq，这个关键点不是很明白，应该是和全局队列做个关联，用于决定运行时机
    _dispatch_retain(tq);
    dq->do_targetq = tq;

    return dq;
}
```

* 每一个用户创建的队列都会指向一个 ***root queue***，这里的指向是说 ***do_targetq*** 指针，就是开始那张图中的的箭头走向；
* 串型队列和并行队列是通过 ***width*** 区分的，串型为1，并行为32766；
* 通过上面的代码可以发现，队列的 ***dq_serialnum*** 是从16开始自增的，其中1-15号队列被系统占用；
* ***do_targetq*** 的意义暂时还不是很明白，等大侠解答。


## dispatch\_sync ##
``` c
void dispatch_sync(dispatch_queue_t dq, dispatch_block_t work) {
    dispatch_function_t func = _dispatch_Block_invoke(work);
    void *ctxt = work;

    // 如果队列宽度为1
    if (dq->dq_width == 1) { // 主线程会从这里走
        dispatch_barrier_sync_f(dq, ctxt, func);
        return;
    }

    if (dq->dq_items_tail) {
        dispatch_thread_event_s event = _dispatch_thread_event_init(&event);
        uint32_t th_self = _dispatch_tid_self();
        struct dispatch_continuation_s dc = dispatch_continuation_s_init(dq);
        uint64_t dq_state = os_atomic_load2o(dq, dq_state, relaxed);
        if (unlikely(_dq_state_drain_locked_by(dq_state, th_self))) {
            DISPATCH_CLIENT_CRASH(dq, "dispatch_sync called on queue already owned by current thread");
        }
        
        // ->_dispatch_queue_push_inline，加在队列尾部，同时wakeup
        _dispatch_continuation_push_sync_slow(dq, &dc);
        
        // 等待任务到达
        _dispatch_thread_event_wait(&event);
    }

    // 入栈，保存现场
    dispatch_thread_frame_s dtf;
    _dispatch_thread_frame_push(&dtf, dq);
    // 直接执行方法
    _dispatch_client_callout(ctxt, func);
    // 出栈，恢复现场
    _dispatch_perfmon_workitem_inc();
    _dispatch_thread_frame_pop(&dtf);
    // 结束，恢复队列状态，wakeup队列
    _dispatch_non_barrier_complete(dq);
}

// 等待任务到达
void _dispatch_thread_event_wait(dispatch_thread_event_t dte) {
    /*......*/
    kern_return_t kr;
    do {
        kr = semaphore_wait(dte->dte_semaphore);
    } while (unlikely(kr == KERN_ABORTED));
    DISPATCH_SEMAPHORE_VERIFY_KR(kr);
    /*......*/
    return;
}
```

* ；
* ；
* ；
* 。


## dispatch\_barrier\_sync ##
``` c
void dispatch_barrier_sync(dispatch_queue_t dq, dispatch_block_t work) {
    dispatch_function_t func = _dispatch_Block_invoke(work);

    dispatch_thread_event_s event = _dispatch_thread_event_init(&event);
    struct dispatch_continuation_s dbss = dispatch_continuation_s_init(dq);
    
    // 主线程 dispatch_sync 的任务必须在主线程执行
    // It's preferred to execute synchronous blocks on the current thread
    // due to thread-local side effects, etc. However, blocks submitted
    // to the main thread MUST be run on the main thread
    if (slowpath(_dispatch_queue_is_thread_bound(dq))) {
        // consumed by _dispatch_barrier_sync_f_slow_invoke
        // or in the DISPATCH_COCOA_COMPAT hunk below
        _dispatch_continuation_voucher_set(&dbsc.dbsc_dc, dq, 0);
        // save frame linkage for _dispatch_barrier_sync_f_slow_invoke
        _dispatch_thread_frame_save_state(&dbsc.dbsc_dtf);
        // thread bound queues cannot mutate their target queue hierarchy
        // so it's fine to look now
        _dispatch_introspection_barrier_sync_begin(dq, func);
    }

    // 检查是否有死锁
    uint32_t th_self = _dispatch_tid_self();
    uint64_t dq_state = os_atomic_load2o(dq, dq_state, relaxed);
    if (unlikely(_dq_state_drain_locked_by(dq_state, th_self))) {
        DISPATCH_CLIENT_CRASH(dq, "dispatch_barrier_sync called on queue already owned by current thread");
    }

    // 加在队列尾部，同时wakeup
    _dispatch_continuation_push_sync_slow(dq, &dc);
    // 等待任务到达
    _dispatch_thread_event_wait(&event);

    // 入栈，保存现场
    dispatch_thread_frame_s dtf;
    _dispatch_thread_frame_push(&dtf, dq);
    // 执行方法
    _dispatch_client_callout(ctxt, func);
    // 出栈，恢复现场
    _dispatch_perfmon_workitem_inc();
    _dispatch_thread_frame_pop(&dtf);

    // 结束，恢复队列状态，wakeup队列
    _dispatch_barrier_complete(dq);
}
```

* ；
* ；
* ；
* 。


## dispatch\_async ##
``` c
void dispatch_async(dispatch_queue_t dq, dispatch_block_t work) {
    //:创建dc并初始化
    dispatch_continuation_t dc = _dispatch_continuation_alloc();
    dc->dc_flags = DISPATCH_OBJ_CONSUME_BIT | DISPATCH_OBJ_BLOCK_BIT;
    dc->dc_ctxt = _dispatch_Block_copy(work);
    dc->dc_func = _dispatch_call_block_and_release;

    if (dq->dq_width > 1) { //:如果队列宽度大于1
        dq = dq->do_targetq;
        // Find the queue to redirect to
        while (dq->dq_width > 1) {
            if (!fastpath(_dispatch_queue_try_acquire_async(dq))) {
                break;
            }
            if (!dou._dc->dc_ctxt) {
                // find first queue in descending target queue order that has an autorelease frequency set, and use that as the frequency for this continuation.
                dou._dc->dc_ctxt = (void *)
                (uintptr_t)_dispatch_queue_autorelease_frequency(dq);
            }
            dq = dq->do_targetq;
        }
    }

    //:更新队列尾部，和dispatch_barrier_async相同->_dispatch_continuation_push(dq, dc);
    if (unlikely(_dispatch_queue_push_update_tail(dq, tail))) { //:如果队列为空，会return ture
        //:如果队列为空，那么更新队尾后队列头部也要更新一下
        _dispatch_queue_push_update_head(dq, tail, true);
        dispatch_wakeup_flags_t flags = DISPATCH_WAKEUP_CONSUME | DISPATCH_WAKEUP_FLUSH;
        //:dx_wakeup 会调用队列的_dispatch_queue_wakeup函数指针，每个队列有自己不同的处理，见do_weakeup.c
        (&dq->do_vtable->_os_obj_vtable)->dx_wakeup(dq, pp, 0)
    }

}
```

* ；
* ；
* ；
* 。


## dispatch\_barrier\_async ##
``` c
void dispatch_barrier_async(dispatch_queue_t dq, dispatch_block_t work) {
    //:创建dc并初始化
    dispatch_continuation_t dc = _dispatch_continuation_alloc();
    dc->dc_flags = DISPATCH_OBJ_CONSUME_BIT | DISPATCH_OBJ_BARRIER_BIT | DISPATCH_OBJ_BLOCK_BIT;
    dc->dc_ctxt = _dispatch_Block_copy(work);
    dc->dc_func = _dispatch_call_block_and_release;

    //:更新队列尾部
    if (unlikely(_dispatch_queue_push_update_tail(dq, tail))) { //:如果队列为空，会return ture
        //:如果队列为空，那么更新队尾后队列头部也要更新一下
        _dispatch_queue_push_update_head(dq, tail, true);
        dispatch_wakeup_flags_t flags = DISPATCH_WAKEUP_CONSUME | DISPATCH_WAKEUP_FLUSH;
        //:dx_wakeup 会调用队列的_dispatch_queue_wakeup函数指针，每个队列有自己不同的处理，见do_weakeup.c
        (&dq->do_vtable->_os_obj_vtable)->dx_wakeup(dq, pp, 0)
    }
}
```

* ；
* ；
* ；
* 。


## do\_weakup ##
``` c
// 宏dx_wakeup的实际调用，dx_wakeup 会调用队列的.do_wakeup函数指针，每个队列有自己不同的处理，这里列举:"queue", "serial-queue", "concurrent-queue", "queue-context"的处理方式

void _dispatch_queue_wakeup(dispatch_queue_t dq, pthread_priority_t pp, dispatch_wakeup_flags_t flags) {

    //:判断dq是否有任务排队，如果有，则将target标记为唤醒
    if (_dispatch_queue_class_probe(dq)) {
        dispatch_queue_wakeup_target_t target = DISPATCH_QUEUE_WAKEUP_TARGET;
        _dispatch_queue_class_wakeup(dq, pp, flags, target); //:唤醒队列执行

    } else {
        return _dispatch_release_tailcall(dq); //:释放dq，引用计数减1
    }
}

void _dispatch_queue_class_wakeup(dispatch_queue_t dq, pthread_priority_t pp, dispatch_wakeup_flags_t flags, dispatch_queue_wakeup_target_t target) {
    uint64_t old_state, new_state = 0;

    os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, relaxed,{
        new_state = old_state;
        if (_dq_state_is_runnable(dq_state) && //:状态是可运行
            !_dq_state_is_enqueued(dq_state) && //:状态不是入队列
            !_dq_state_drain_locked(dq_state)) { //:状态不是被锁
            new_state |= DISPATCH_QUEUE_ENQUEUED;
        } else {
            os_atomic_rmw_loop_give_up(break);
        }
    });

    if ((old_state ^ new_state) & DISPATCH_QUEUE_ENQUEUED) { //:如果新出现了DISPATCH_QUEUE_ENQUEUED标记
        _dispatch_queue_class_wakeup_enqueue(dq, pp, flags, target);

    } else {
        _dispatch_release_tailcall(dq);
    }
}
```

* ；
* ；
* ；
* 。


## do\_invoke ##
``` c
// 宏dx_invoke的实际调用，dx_invoke 会调用队列的.do_invoke函数指针，每个队列有自己不同的处理，这里列举:"queue", "serial-queue", "concurrent-queue", "main-queue", "runloop-queue", "queue-context"的处理方式


#pragma mark - _dispatch_queue_invoke
void _dispatch_queue_invoke(dispatch_queue_t dq, dispatch_invoke_flags_t flags) {
    struct dispatch_object_s *dc = NULL;
    dispatch_queue_t tq = NULL;

    uint64_t dq_state = 0;
    uint64_t to_unlock = _dispatch_queue_drain_try_lock(dq, flags, &dq_state);
    if (likely(to_unlock)) {

    drain_pending_barrier:
        /*如果是barrier的情况，主要是检查并重置一下flags和priority*/

    attempt_running_slow_head:

        //:真正分发队列任务的地方
        tq = dispatch_queue_invoke2(dq, flags, &to_unlock, &dc);

        //:如果返回的tq为空，但是dq释放锁失败，会被认为是dirty的情况，尝试再处理一次
        if (!tq && !_dispatch_queue_drain_try_unlock(dq, to_unlock)) {
            goto attempt_running_slow_head;
        }
    }

    //:如果执行完后还返回来了一个tq和dc，那么把这个任务推迟执行
    if (tq && dc) {
        return _dispatch_queue_drain_deferred_invoke(dq, flags, to_unlock, dc);
    }

    //:如果返回了tq，那么检查一堆状态，看看需不需要再次进行操作
    if (tq) {
        uint64_t old_state, new_state;
        /*检查一堆tq的状态，主要是 DISPATCH_QUEUE_IN_BARRIER，DISPATCH_QUEUE_PENDING_BARRIER，DISPATCH_QUEUE_ENQUEUED 状态的检查*/

        //:如果在barrier状态中
        if (_dq_state_is_in_barrier(new_state)) {
            to_unlock &= DISPATCH_QUEUE_ENQUEUED;
            to_unlock += DISPATCH_QUEUE_IN_BARRIER;
            goto drain_pending_barrier;
        }
        //:如果需要再次入队列，则push，等待执行
        if ((old_state ^ new_state) & DISPATCH_QUEUE_ENQUEUED) {
            return _dispatch_queue_push(tq, dq, 0);
        }
    }

    //:最后释放dq，返回
    return _dispatch_release_tailcall(dq);
}


#pragma mark - dispatch_queue_invoke2
/// dispatch_queue_invoke2//
static inline dispatch_queue_t dispatch_queue_invoke2(dispatch_queue_t dq, dispatch_invoke_flags_t flags, uint64_t *owned, struct dispatch_object_s **dc_ptr) {
    if (dq->dq_width > 1) { //:并发队列
        return _dispatch_queue_drain(dq, flags, owned, dc_ptr, false);

    } else { //:串型队列
        flags &= ~(dispatch_invoke_flags_t)DISPATCH_INVOKE_REDIRECTING_DRAIN;
        return _dispatch_queue_drain(dq, flags, owned, dc_ptr, true);
    }
}


#pragma mark - _dispatch_queue_drain
/// _dispatch_queue_drain//
static dispatch_queue_t _dispatch_queue_drain(dispatch_queue_t dq, dispatch_invoke_flags_t flags, uint64_t *owned_ptr, struct dispatch_object_s **dc_out, bool serial_drain) {

    struct dispatch_object_s *dc = NULL, *next_dc;
    uint64_t owned = *owned_ptr;
    if (_dq_state_is_in_barrier(owned)) {
        owned = DISPATCH_QUEUE_IN_BARRIER;
    }

    //:切换到执行线程
    dispatch_thread_frame_s dtf;
    _dispatch_thread_frame_push(&dtf, dq);

    while (dq->dq_items_tail) {
        //:获得队首的任务
        dc = _dispatch_queue_head(dq);
        do {
            //:如果是(串型队列||barrier任务)
            if (serial_drain || _dispatch_object_is_barrier(dc)) {
                //:如果不是串型队列，并且当前不在barrier处理中
                if (!serial_drain && owned != DISPATCH_QUEUE_IN_BARRIER) {
                    goto out;
                }

                //:进行一次pop操作，将head任务取出来
                next_dc = _dispatch_queue_next(dq, dc);

            } else {
                //:进行一次pop操作，将head任务取出来
                next_dc = _dispatch_queue_next(dq, dc);

                //:将任务发到其他线程执行
                if (flags & DISPATCH_INVOKE_REDIRECTING_DRAIN) {
                    _dispatch_continuation_redirect(dq, dc);
                    continue;
                }
            }

            //:执行任务
            _dispatch_continuation_pop_inline(dc, dq, flags);

            //:如果线程被推迟
            if (unlikely(dtf.dtf_deferred)) {
                goto out_with_deferred_compute_owned;
            }

        } while ((dc = next_dc));
    }

out:
    //:退出执行线程
    _dispatch_thread_frame_pop(&dtf);
    return dc ? dq->do_targetq : NULL;

out_with_deferred_compute_owned:
    /*一堆保留现场的操作*/
    _dispatch_thread_frame_pop(&dtf);
    return dq->do_targetq;
}
```

* ；
* ；
* ；
* 。


---
未完，明天继续...