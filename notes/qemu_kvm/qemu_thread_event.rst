QEMU thread and event model
===========================

- documents

  - [2011] `QEMU Internals: Overall architecture and threading model <http://blog.vmsplice.net/2011/03/qemu-internals-overall-architecture-and.html>`_
  - [2015] `Improving the QEMU Event Loop <http://www.linux-kvm.org/images/6/64/03x08-Aspen-Fam_Zheng-Improving_the_QEMU_Event_Loop.pdf>`_
  - [2014] `docs/devel/multiple-iothreads.txt <https://lxr.missinglinkelectronics.com/qemu+v2.10.0/docs/devel/multiple-iothreads.txt>`_ (at docs/multiple-iothreads.txt before QEMU 2.10)

architecture: Event-driven multi-threaded

- event loop used for timer, fd, monitoring ... etc

  - non-blocking IO
  - callback or coroutines

- multi-thread architecture but with big lock

  - vcpu threads execute in parallel
  - specific task runs in other threads: RAM live migration, remote displaying encoding, virtio-blk dataplane
  - global mutex

doc1
----

[2011] `QEMU Internals: Overall architecture and threading model <http://blog.vmsplice.net/2011/03/qemu-internals-overall-architecture-and.html>`_

a. event-driven core
  
   - ``main_loop_wait()``

     a. ``set_fd_handler()``
     b. run expired timer
     c. run BH(bottom halves)

   - callback: 
   
     - core execution is sequentially and atomicly. only one thread.
     - no blocking system call.

b. Offloading specific tasks to worker threads

   - example: posix-aio-compat.c, ui/vnc-jobs-async.c
   - core isn't thread-safe, worker threads can't call core => workers communicate core by adding ``qemu_eventfd()`` or pipe to event-loop

c. Executing guest code

   - TCG or KVM mode
   - signal interrupt guest code and return qemu core.

d. iothread architecture

   - truly SMP in KVM mode
   - single qemu thread run guest code and event loop => vcpu thread run guest code and iothread run event loop
   - Qemu Core code is still not thread-safe => vcpu thread and iothread hold a global mutex.
   - but, enter guest code and wait for event loop doesn't hold lock: most of time thread doesn't hold mutex.

doc2
----

[2015] `Improving the QEMU Event Loop <http://www.linux-kvm.org/images/6/64/03x08-Aspen-Fam_Zheng-Improving_the_QEMU_Event_Loop.pdf>`_

- 請儘量將 slide 第一部份 "The event loops in QEMU" 研究完. 
  virtio guest driver 在呼叫 qemu backend 會用到 aio, 最低限度也要了解 aio 跟 slirp 的部份.

  a. QEMU multithreading: vCPU, iothread, pool workers ...
  b. Main loop in front: prepare, poll, dispatch
  c. Main loop under the surface: slirp, iohandler, glib
  d. GSource: chardev, aio context

