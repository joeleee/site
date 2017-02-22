---
title: 扒了扒libdispatch源码
date: 2017-02-21 15:47:10

categories:
- Objective-C

tags:
- libdispatch
- GCD
---

前几天扒了一下libdispatch的源码，当前最新的是 libdispatch-703.30.5.tar

``` c
void _dispatch_thread_event_wait_slow(dispatch_thread_event_t dte) {
    kern_return_t kr;
    do {
        kr = semaphore_wait(dte->dte_semaphore);
    } while (unlikely(kr == KERN_ABORTED));
    DISPATCH_SEMAPHORE_VERIFY_KR(kr);
    return;
}
```