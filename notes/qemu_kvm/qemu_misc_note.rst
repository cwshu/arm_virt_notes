QEMU/KVM architecture overview
==============================

it's a note of "KVM Arch" slide in 2015 KVM forum

- [2015] `KVM Architecture Overview <https://vmsplice.net/~stefan/qemu-kvm-architecture-2015.pdf>`_
- old version, `QEMU Code Overview <https://vmsplice.net/~stefan/qemu-code-overview.pdf>`_

  - with some source code hint

Qemu process model (Qemu and Guest OS)
--------------------------------------

- linux userspace process: process memory = qemu memory + guest OS physical memory
- each KVM vCPU is a thread.

  - Host kernel scheduler decides when vCPUs run

Event-driven multi-threaded
---------------------------

- event loop used for timer, fd, monitoring ... etc

  - non-blocking IO
  - callback or coroutines

- multi-thread architecture but with big lock

  - vcpu threads execute in parallel
  - specific task runs in other threads: RAM live migration, remote displaying encoding, virtio-blk dataplane
  - global mutex

see :doc:`./qemu_thread_event` for detail

host/guest device emulation split
---------------------------------

- guest device: device model visible to guest
- host device: performs IO on behalf of guest

use QEMU CLI option as example.

- There are 2 parts to networking in QEMU

  a. virtual network device: ``-device <device>,netdev=<id>``
  b. network backend: ``-netdev <backend>,id=<id>``

- 2 examples

  ::

    # guest device=e1000 & host device=user (user network/slirp)
    -netdev user,id=net0 \
    -device e1000,netdev=net0

    # guest device=virtio-net & host device=user
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0

  

virtio devices
--------------

- KVM implements virtio device models

  - http://docs.oasis-open.org/virtio/virtio/v1.0/cs04/virtio-v1.0-cs04.html#x1-20001

- Red Hat contributes to Linux and Windows guestdrivers
- see :doc:`virtio` for detail

vhost in-kernel devices
-----------------------

- vhost drivers emulate virtio devices in host kernel for better performance

  - vhost_net.ko: high-performance virtio-net emulation, kernel-only zero-copy and interrupt handling features.

Qemu

- `QEMU Source Code Study - 1 <http://sdytlm.blogspot.tw/2013/10/test.html>`_
