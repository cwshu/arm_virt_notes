vhost net
=========

- document

  - `KVM-Architecture-LK2010 slide p.18 <http://www.linux-kongress.org/2010/slides/KVM-Architecture-LK2010.pdf>`_
  - `Stefan Hajnoczi - QEMU Internals: vhost architecture <http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.htm>`_
  - `gaowanlong - KVM irqfd and ioeventfd <http://blog.allenx.org/2015/07/05/kvm-irqfd-and-ioeventfd>`_
  
    - `ioeventfd patch <http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=d34e6b175e61821026893ec5298cc8e7558df43a>`_
    - `irqfd patch <http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=721eecbf4fe995ca94a9edec0c9843b1cc0eaaf3>`_
  
  - `gaowanlong - vhost architecture <http://blog.allenx.org/2013/09/09/vhost-architecture>`_
  - `RoyLuo's Notes - vhost architecture <http://royluo.org/2014/08/22/vhost/>`_
  - `virtio 后端方案 vhost <http://blog.csdn.net/majieyue/article/details/51262510>`_
  - [PATCHv9 0/3] vhost: a kernel-level virtio server <https://www.spinics.net/lists/kvm/msg25348.html>

- vhost_net.ko

  - Host user space opens and configures kernel helper
  - virtio as guest-host interface
  - KVM interface: eventfd

    - TX trigger => ioeventfd
    - RX signal  => irqfd

  - Linux interface via tap or macvtap
  - future: multi-queue vhost-net

`Stefan Hajnoczi - QEMU Internals: vhost architecture <http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html>`_

- overview

  - vhost driver provide in-kernel virtio devices for KVM.
    compare to QEMU, it can directly call into kernel subsystem without system call.
  - vhost-net driver emulate virtio network card in the host kernel. (``driver/vhost/``)

- driver model

  - ``/dev/vhost-net``, QEMU open and init it if parameter has ``-netdev tap,vhost=on``
  - vhost-worker-thread (``vhost-<pid>``): (1) handle IO event (2) perform device emulation

    - create at initialization time.

- in-kernel virtio emulation
  
  - vhost doesn't emulate complete virtio PCI adapter, it restrict itself to virtqueue ops only.
  - vhost worker thread do IO:
  
    - wait virtqueue kick, recieve packet from tx virtqueue.
      transmit over tap fd.
    - do fd polling, wake up when packets come in over the tap fd.
      place packets to rx virtqueue

- vhost as a userspace interface

  - guest kick host 時, 需要有個方法通知 vhost worker thread.
  - virtio 不依賴 KVM module. 所以要透過 eventfd 來通知.
  - ioeventfd:
  - irqfd: allow eventfd to trigger guest interrupts

