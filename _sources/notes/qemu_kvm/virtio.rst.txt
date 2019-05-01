Virtio
======

- intro

  - virtio devices are implemented over MMIO, Channel I/O and PCI bus transports
  - virtqueue, vring

Document
--------

- initial paper from Rusty Russel: `(Rusty, 2008) virtio: Towards a De-Facto Standard For Virtual I/O Devices <https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf>`_
- about virtio spec:

   - `Virtio spec from OASIS <http://docs.oasis-open.org/virtio/virtio/v1.0/cs04/virtio-v1.0-cs04.html#x1-20001>`_
   - `(2014), Standardizing virtio [LWN.net] <https://lwn.net/Articles/580186/>`_

- `(2010), IBM developer works, Virtio: An I/O virtualization framework for Linux <http://www.ibm.com/developerworks/library/l-virtio/>`_
- `CMU 15-412 materials, Virtio: An I/O virtualization framework for Linux <https://www.cs.cmu.edu/~412/lectures/Virtio_2015-10-14.pdf>`_
- `(2014, devconf.cz), Stefan Hajnoczi. VIRTIO 1.0 - Paravirtualized I/O for KVM and beyond <https://vmsplice.net/~stefan/virtio-devconf-2014.pdf>`_

- Performance comparsion: `virtio-blk latency <http://www.linux-kvm.org/page/Virtio/Block/Latency>`_

notes
~~~~~

- paper note: :doc:`/notes/qemu_kvm/virtio_paper`
- virtqueue initialization (e.g. config)
- virtqueue 5 ops 簡介

virtio-blk
----------

- documents

  - `virtio blk 流程 (p.18 to 21) <https://vmsplice.net/~stefan/qemu-kvm-architecture-2015.pdf>`_
  - `(2012) virtio blk performance improvement <http://www.linux-kvm.org/images/f/f9/2012-forum-virtio-blk-performance-improvement.pdf>`_
  - `Virtio-Blk性能加速方案 <http://royluo.org/2014/08/31/virtio-blk-improvement/>`_

- bio based virtio-blk guest driver: remove IO scheduling.
- vhost blk host driver: remove host fs layer, no qemu userspace.

Misc
----
- virtqueue kick: a pio write to a virtio PCI hardware register

  - Programming IO: https://en.wikipedia.org/wiki/Programmed_input/output
  - http://www.linux-kvm.org/page/Virtio/Block/Latency
  - 好吧, 看來 virtqueue kick 就是 trap, vring interrupt 也是一般的 interrupt.

- `virtio-ioeventfd (userspace emulation) <http://qemu-project.org/Features/VirtioIoeventfd>`_

  - `mailing list discussion <https://lists.nongnu.org/archive/html/qemu-devel/2011-01/msg02492.html>`_
  - `patch <https://patchwork.ozlabs.org/patch/70806/>`_
  - virtio-pci: Use ioeventfd for virtqueue notify
