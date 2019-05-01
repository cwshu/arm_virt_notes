IRQFD
=====

IRQFD: eventfd handled at kernel, kernel use ``wait_queue`` and ``kvm_set_irq()`` to inject IRQ to VM.

APIs::

  - KVM VM ioctl - KVM_IRQFD
  - KVM VM ioctl - KVM_IOEVENTFD
  - QEMU kvm-all.c - kvm_irqchip_assign_irqfd()
  - QEMU kvm-all.c - kvm_set_ioeventfd_mmio()
  - QEMU kvm-all.c - kvm_set_ioeventfd_pio()

source analysis
---------------

::
    
    kvm_arch_set_irq_inatomic()

    // workqueue 
    work_struct
    INIT_WORK()
    schedule_work()
    flush_work()
    queue_work()
    
    // wait queue
    wait_queue_t
    init_waitqueue_func_entry()

    eventfd_ctx_remove_wait_queue()

wait queue: irqfd_wakeup()

    POLLIN: run workqueue irqfd->inject
    POLLHUP: irqfd_deactivate()

work queue: irqfd_inject()

    kvm_set_irq()
    IRQ resampler

work queue: irqfd_shutdown()
    
    remove evenfd from wait_queue
    flush_work() irqfd->inject


reference
---------

- `gaowanlong - KVM irqfd and ioeventfd <http://blog.allenx.org/2015/07/05/kvm-irqfd-and-ioeventfd>`_

  - `ioeventfd patch <http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=d34e6b175e61821026893ec5298cc8e7558df43a>`_
  - `irqfd patch <http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=721eecbf4fe995ca94a9edec0c9843b1cc0eaaf3>`_

