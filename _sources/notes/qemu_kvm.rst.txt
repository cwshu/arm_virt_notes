QEMU/KVM note
=============

QEMU Basic
----------

   .. toctree::
      :maxdepth: 2

      qemu_kvm/qemu_init
      qemu_kvm/QOM1
      qemu_kvm/qemu_glib

Memory
------

   .. toctree::
      :maxdepth: 2

      qemu_kvm/qemu_memory
      qemu_kvm/qemu_memory_kml
      qemu_kvm/system_memory

ivshmem
-------

   .. toctree::
      :maxdepth: 2

      qemu_kvm/ivshmem
      qemu_kvm/ivshmem_memory
      qemu_kvm/qemu_ivshmem

QEMU Net
--------
    
   .. toctree::
      :maxdepth: 2

      qemu_kvm/qemu_net

   virtual network devices

   .. toctree::
      :maxdepth: 2

      qemu_kvm/e1000_device
      qemu_kvm/virtio_net

   network backends (尚缺 tap network)

   .. toctree::
      :maxdepth: 2

      qemu_kvm/slirp
   
   in-kernel emulation: :doc:`vhost-net <qemu_kvm/vhost_net>`, :doc:`irqfd source analysis <qemu_kvm/irqfd>`

QEMU Misc
---------

   .. toctree::
      :maxdepth: 2

      qemu_kvm/qemu_misc_note
      qemu_kvm/qemu_thread_event

KVM
---

   .. toctree::
      :maxdepth: 2

      qemu_kvm/run_and_exit
      qemu_kvm/oenhen_part2_3
      qemu_kvm/kvm_resources
      qemu_kvm/tracepoint
