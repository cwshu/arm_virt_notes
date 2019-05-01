ivshmem source code
===================

- [hw/misc/ivshmem.c] Inter-VM shared memory

- class name: TYPE_IVSHMEM
- inheritance: TYPE_IVSHMEM -> TYPE_IVSHMEM_COMMON -> TYPE_PCI_DEVICE -> TYPE_DEVICE

- TYPE_IVSHMEM members (from class init)

  - k->realize = ivshmem_realize
  - dc->desc = "Inter-VM shared memory (legacy)"
  - dc->props = ivshmem_properties

- TYPE_IVSHMEM_COMMON members (from class init)

  - k->realize = ivshmem_common_realize
  - dc->desc = "Inter-VM shared memory"

  - k->exit = ivshmem_exit
  - k->config_write = ivshmem_write_config
  - dc->reset = ivshmem_reset

Misc
----
::
    
    static void ivshmem_IntrMask_write(IVShmemState *s, uint32_t val)

    1. IntrMask read/write
    2. IntrStatus read/write

    3. RW interrupt register, 如果有寫入, 則透過 pci update irq

    static void ivshmem_io_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)

    1. io_read/write
    2. MemoryRegionOps

    ivshmem_vector_notify

    1. notify, mask/unmask, poll

    static void watch_vector_notifier(IVShmemState *s, EventNotifier *n, int vector)

    1. int eventfd = event_notifier_get_fd(n);
    2. qemu_set_fd_handler(eventfd, ivshmem_vector_notify, NULL, &s->msi_vectors[vector]);

    static void ivshmem_add_eventfd(IVShmemState *s, int posn, int i)

    static void process_msg_shmem(IVShmemState *s, int fd, Error **errp)

    1. process msg connect/disconnect
    2. process msg 
