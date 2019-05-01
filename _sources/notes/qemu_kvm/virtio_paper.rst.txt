virtio-paper
============

https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf

abstract
--------
- virtio: a series of efficient, well-maintained Linux drivers

  - can be adapted for various hypervisor using a shim layer.

- vring: ring buffer transport implementation
- provide an implementation

  - presents (1) vring transport (2) device config as a PCI device
  - means:

    - guest OS merely need a new PCI driver
    - hypervisor only need add vring support to virtual device (they implement)

- my idea

  - 通用的 (generic) IO PV protocol, 高相容性
  
    - 向下相容: 不同的 hypervisor 可以使用同一套 IO PV protocol 跟 guest driver 溝通. 如同不同 hypervisor 都使用相同硬體介面.
    - 向上相容: [?] 跟原生的 kernel driver 儘量相似, 希望只要改掉硬體介面即可 porting 成 virtio driver.

  - protocol 組成
  
    - 實作必須具有 IO PV 的必備要素, 比如說 transport buffer, vring 便用於此
    - [?] virtio 同時是 protocol, 也是 API layer. 這讓我想到 remote procedure call

- my idea2

  - design 目標 
  - API (config + transport)
  - 應用的 driver, 需要特殊處理的應用: PCI driver
  - 效能分析

  - 目前有使用的實作
  - 未來的擴展

Table Of Content
----------------
- 1. intro
- 2. virtio: the three goals
- 3. virtio: a Linux internal abstraction API

  - Virtqueues: a transport abstraction

- 4. virtio_ring: a transport implementation for virtio

  - 4.1. a note on Zero-Copy and religion of page flipping

- 5. current virtio drivers

  - 5.1. virtio block driver
  - 5.2. virtio network driver 

- 6. virtio_pci: a PCI implementation of vring and virtio
- 7. performance
- 8. adoption
- 9. future work
- 10. conclusions

Content
-------

1. intro
~~~~~~~~
通用的 IO Para-Virt Protocol, 跨 hypervisor/guest VM.

2. virtio: the three goals
~~~~~~~~~~~~~~~~~~~~~~~~~~

- 3 goals
  
  a. driver unification
  b. uniformity to provide common ABI for (1) general publication (2) use of buffer.
  c. device probing and config


- 如果 developer 熟悉 Linux, 他們可能會直接 map Linux API 在 (virtual IO mechanism?) ABI 上面.
- cross-device: common ABI for (1) general publication (2) use of buffer.
- provide a two complete ABI implementations

  - using (1) virtio_ring infrastructure (2) Linux API for virtual IO device.
  - implement final part of virtual IO device: device probe and config

- explicit seperate (1) driver (2) transport (3) config

  - 反例: 如果其他 hypervisor 要用 Xen Linux network driver, 必須 support Xenbus probing and config system

3. virtio: a Linux internal abstraction API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 4 parts of config op 

  a. RW feature bits
  b. RW config space
  c. RW status bits
  d. device reset

- feature bit
  
  - device feature: e.g. VIRTIO_NET_F_CSUM is checksum offload feature of net device
  - device specific

- config space

  - when device feature VIRTIO_NET_F_MAC is set
  - MAC address of device is in config space

- RW 8 bits status word

  - indicate device probe
  - VIRTIO_CONFIG_S_DRIVER_OK is set means that host knows what feature it understands and wants to use.

Transport Abstraction: Virtqueue

- virtio-blk has one queue
- virtio-net/console has two queues, input/output both use one.

- each buffer is a scatter-gather array
- virtqueue op struct: 5 ops

  - add_buf
  - kick
  - get_buf
  - enable/disable_cb

- data exchange flow: add_buf() => kick host() => host update data => get_buf()
- 5 ops

  - disable_cb is a hint that guest doesn't want to know "when the buffer is used" => same as disable interrupt.

4. virtio_ring: a transport implementation for virtio
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- virtio ring consist of 3 parts

  - descriptor array: (addr, length) pair chain.
  - avail ring: guest indicate. chain is ready to use.
  - used ring: host indicate. chain is used.

- descriptor array:
  
  - (addr, length)
  - optional next
  - flags: 2 bits, 1 for RW, 1 for next option

- used ring 有故意跟 available ring/descriptor array 放在不同 page. 這樣 cache 的表現會比較好.

- interrupt suppression flag

  - available ring 跟 used ring 都有.
  - for optimization (virtqueue kick (vmexit/trap) and interrupt completion)
  - available ring 是 guest 用來通知 host 不用送 completion interrupt (disable_cb)
  - used ring 是 host 用來通知 guest 不用 virtqueue kick host.
  - optimization example?

4.1. a note on Zero-Copy and religion of page flipping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- efficient IO need 2 things

  a. the number of notification per op. => by virtio_ring interrupt suppression flag.
  b. amount of cache-cold data which is accessed.

- Zero-Copy
- page flipping

5. current virtio drivers
~~~~~~~~~~~~~~~~~~~~~~~~~

- virtio block driver: single request queue

  - first 16 byte is RO header = (type, ioprio, sector)
  
    - 4 kinds of type: R, W, SCSI command, W Barrier
    - IO priority hint
    - sector: 512 bytes offset
  
  - SCSI command: e.g. (a) eject virtual CDROM. (b) implement SCSI HBA over virtio.

- virtio network driver

  - 2 virtqueue: transmission and receiving virtqueue.
  - Virtual HW set large MTU. It reduce the number of hypercall.
    
    - large MTU means few PCI transfer to card.
    - In virtual env, it means fewer numbers of calls out from virtual env.

  - guest can set interrupt suppression flag for transmission virtqueue. guest doesn't care when transmission is finish.
    
    - The only exception is when the queue is full.

6. virtio_pci: a PCI implementation of vring and virtio
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

7. performance
~~~~~~~~~~~~~~
