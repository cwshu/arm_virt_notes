Virtio Network
==============

:doc:`Virtio note </notes/qemu_kvm/virtio>`

virtio-net qemu device
----------------------

from QEMU 2.9 source code.

MMIO and virtqueue_kick()
~~~~~~~~~~~~~~~~~~~~~~~~~

``virtqueue_kick()`` is a MMIO operation, and it triggers virtio-device to output data from driver.
Output interface is ``VirtQueue->handle_output()`` or ``VirtQueue->handle_aio_output()``.
Virtio-net use output interface to send packet (TX path).

::

    // virtio_mmio_write() => virtio_queue_notify() => vq->{handle_output,handle_aio_output}()

    // pseudocode
    void virtio_mmio_write(void *opaque, hwaddr offset, uint64_t value, unsigned size)
        VirtIOMMIOProxy *proxy = (VirtIOMMIOProxy *)opaque;
        VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);

        if offset == VIRTIO_MMIO_QUEUE_NOTIFY:
            vdev->vq[value]->handle_output(vdev, vdev->vq[value])
    
    // handle_output interface
    void handle_output(VirtIODevice *, VirtQueue *);
    bool handle_aio_output(VirtIODevice *, VirtQueue *);

virtio-mmio device handles MMIO by ``VirtIOMMIOProxy``'s ``virtio_mmio_write()``.

virtio driver does ``virtqueue_kick()`` when MMIO offset(guest address) is ``VIRTIO_MMIO_QUEUE_NOTIFY``.
device handles ``virtqueue_kick()`` by ``virtio_queue_notify()``.

In ``virtio_queue_notify()``, device handles output of one virtqueue. 
MMIO value assigns which virtqueue should be used.
each virtqueue has it's ``handle_output()`` or ``handle_aio_output()`` callback.

virtio device can use ``virtio_add_queue()`` to set ``VirtQueue->handle_output``, and use ``virtio_queue_aio_set_host_notifier_handler()`` to set ``VirtQueue->handle_aio_output``.

Virtio TX
~~~~~~~~~

virtio-net handles TX by timer or QEMU BH. We use BH as example::

    void virtio_net_add_queue(VirtIONet *n, int index){
        if( ... ){
            n->vqs[index].tx_vq = virtio_add_queue(vdev, 256, virtio_net_handle_tx_timer);
            n->vqs[index].tx_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, virtio_net_tx_timer, &n->vqs[index]);
        }
        else{
            n->vqs[index].tx_vq = virtio_add_queue(vdev, 256, virtio_net_handle_tx_bh);
            n->vqs[index].tx_bh = qemu_bh_new(virtio_net_tx_bh, &n->vqs[index]);
        }
    }

.. image:: img/virtio.png

``virtio_net_handle_tx_bh()`` handles MMIO, and async calls BH.

Async BH callback ``virtio_net_tx_bh()``::

    virtio_net_tx_bh() => virtio_net_flush_tx()

real TX ``virtio_net_flush_tx()``

- guest driver 傳輸的資料放在 ``VirtQueue`` 內 ``q->tx_vq``
- virtio device 每次會拿一個 ``VirtQueue`` 的 element 來做傳輸.

  - ``virtqueue_pop()``: 取一個 element
  - emulate device 傳輸: 呼叫 ``qemu_send_packet()`` 系列 API.

- completion interrupt
- net_hdr_swap, host_hdr_len 

Virtio API
~~~~~~~~~~

::

    TYPE_VIRTIO_NET => TYPE_VIRTIO_DEVICE => TYPE_DEVICE
    
    - VirtIONet => VirtIODevice => DeviceState
    - X => VirtioDeviceClass => DeviceClass

    struct VirtIONet {
        VirtIODevice parent_obj

        VirtIONetQueue *vqs
        VirtQueue *ctrl_vq

        virtio_net_conf net_conf
        NICState *nic
        NICConf nic_conf
    }

    struct VirtIONetQueue {
        VirtQueue *rx_vq
        VirtQueue *tx_vq

        QEMUTimer *tx_timer
        QEMUBH *tx_bh
    }

VirtIODevice, VirtQueue and VRing::
    
    hw/virtio/virtio.c
    include/hw/virtio/virtio.h

    struct VirtIODevice {
        VirtQueue *vq
        uint8_t isr
    }

    struct VirtQueue {
        VRing *vring
        VirtIOHandleOutput
        VirtIOHandleAIOOutput
        EventNotifier guest_notifier, host_notifier;
    }

    struct VRing {
        hwaddr desc, avail, used
        VRingMemoryRegionCaches *caches // rcu + 3 MemoryRegionCache (desc, avail, used)
    }

virtqueue API::
    
    elem = virtqueue_alloc_element(sz, out_num, in_num);
    virtqueue_detach_element(vq, elem, len);

    elem = virtqueue_pop()                
    virtqueue_unpop(q->rx_vq, elem, total);
    virtqueue_fill(): vring_used_write()
    virtqueue_flush(): vq->used_idx += count
    virtqueue_push(): 先 virtqueue_fill() 一個 element, 然後 virtqueue_flush() 1 格. 

    virtio_notify(vdev, q->rx_vq): 透過 qemu_set_irq() inject VIRQ 給 guest
    virtio_queue_set_notification(): 開關 VRingUsed 的 interrupt suppression.
    virtqueue_read_next_desc()

    virtqueue_map_desc()
    //
    elem = virtqueue_pop()                      => dma_memory_map()   => address_space_map()
    virtqueue_unpop(q->rx_vq, elem, total);     => dma_memory_unmap() => address_space_unmap()

virtio_notify() has 2 backends::

    virtio_notify()

    - VirtIODevice->isr &= value
    - k->notify(); where k (VirtioBusClass is parent bus of VirtIODevice

    k->notify() has 2 kinds

    - virtio_mmio_update_irq()
    - virtio_pci_notify()


進入正題 ``virtqueue_pop()``::

    // 一開始先從現在 VRing 的 head 讀出一個 descriptor
    i = head;
    vring_desc_read(vdev, &desc, desc_cache, i);

    // indirect descriptor 的情況: 需要新的 MemoryRegionCache 以及從 VRing 再讀一個 descriptor
    if (desc.flags & VRING_DESC_F_INDIRECT) {
        len = address_space_cache_init(&indirect_desc_cache, vdev->dma_as, desc.addr, desc.len, false);
        vring_desc_read(vdev, &desc, desc_cache, i);
    }

    // Collect all the descriptors (VRingDesc is a linked-list)
    do {
        map_ok = virtqueue_map_desc(vdev, &out_num, addr, iov, VIRTQUEUE_MAX_SIZE, false, desc.addr, desc.len);
        rc = virtqueue_read_next_desc(vdev, &desc, desc_cache, max, &i);
    } while(rc == VIRTQUEUE_READ_DESC_MORE)


    // 將 VRing 複製出的資料包裝成 VirtQueueElement
    elem = virtqueue_alloc_element(sz, out_num, in_num);
    for(){
        elem->out_addr[i] = addr[i];
        elem->out_sg[i] = iov[i];
    }
    ...

    return elem;

VRingMemoryRegionCache::

    virtio_init_region_cache(vdev, n):
      初始化 VDev 裡的第 n 個 VQ 的 VR 的 VRingMemoryRegionCaches 的 MemoryRegionCache
      使用 address_space_cache_init()
      初始化參數 VR.desc, get_desc_size(vdev), vdev->dma_as
    virtio_queue_update_rings(): 放好 VR.{desc, avail, used} 的 hwaddr, 然後用 virtio_init_region_cache() 初始化 MemoryRegionCache

low level VRing access API::

    vring_desc_read()
        translate endianness:
        read from MemoryRegionCache (addr=i*sizeof(VRingDesc), len=sizeof(VRingDesc))

    vring_used_write()
        translate endianness: virtio_tswap32s()
        write a VRingUsedElem (uelem, sizeof(VRingUsedElem)) 
          to VRingMemoryRegionCaches (&caches->used, pa=offsetof(VRingUsed, ring[i]), len=sizeof(VRingUsedElem)),
          using address_space_write_cached()
        cache invalidate: address_space_cache_invalidate()

    // 同類型 API
    vring_desc_read(): 讀取 VR 的第 n 個 VRingDesc
    vring_avail_ring(): 讀取 VR 的 VRingAvail 的第 i 個 ring
    vring_avail_flags(): 讀取 VR 的 VRingAvail 的 flags
    vring_avail_idx(): 讀取 VR 的 VRingAvail 的 idx
    vring_get_used_event(): vring_avail_ring(vq, vq->vring.num);

    vring_used_write(): 寫入 VR 的 VRingUsed 的第 i 個 VRingUsedElem
    vring_used_idx(): 讀取 VR 的 VRingUsed 的 idx
    vring_used_idx_set(): 寫入 ... (同上)
    vring_used_flags_set_bit(vq, mask): VR's VRingUsed's flags |= mask
    vring_used_flags_unset_bit(): 與上面相反

    vring_set_avail_event()

virtio-access API
"""""""""""""""""

- ``virtio_tswap<size>s()`` 使用 ``bswap()`` API 轉換 endianness
- ``virtio_ld_phy()``: 使用 ``address_space_read()`` 讀取 guest memory
- ``virtio_ld_phy_cached()``: 使用 ``address_space_read()`` 讀取 guest memory，並且有 ``MemoryRegionCache`` 的加速

files::

    virtio-access.h: 定義對 VRing shared memory 的 access API
    memory_ldst.inc.c (exec.c): 
      address_space_ldl_internal() 實作相近 address_space_read()
      address_space_ldl_internal_cached() 實作相近 address_space_read(), 並使用 MemoryRegionCache 的版本.
    bswap.h: 進行 big-endianness 跟 little-endianness 之間的轉換

address_space APIs
""""""""""""""""""

::

    address_space_cache_init()
    address_space_read_cached()

iov series APIs
~~~~~~~~~~~~~~~
- iov_copy():: 

    unsigned iov_copy(
        struct iovec *dst_iov, unsigned int dst_iov_cnt,
        const struct iovec *iov, unsigned int iov_cnt,
        size_t offset, size_t bytes) 

  - copy from ``(start=iov+offset, len=bytes)`` to ``(start=dst_iov, len=bytes)``
  - shallow copy => ``iov`` and ``dst_iov`` use same ``iov_base`` pointer
  - return copied ``iov_cnt``

- iov_to_buf()::

    size_t iov_to_buf(const struct iovec *iov, const unsigned int iov_cnt,
        size_t offset, void *buf, size_t bytes)

  - copy from ``(start=iov+offset, len=bytes)`` to ``(start=buf, len=bytes)``
  - deep copy
  - return copied length

virtio-net kernel driver
------------------------

from Linux Kernel 3.14.

virtqueue, vring, and virtio_device
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

source code::
    
    include/uapi/linux/virtio_ring.h: // struct vring
    include/linux/virtio.h:           // struct virtqueue
    driver/virtio/virtio_ring.c       // struct vring_virtqueue

- ``include/uapi/linux/virtio_*.h`` are virtio ABIs between guest and host.


::
    
    struct vring_virtqueue {
        struct virtqueue vq;
        struct vring vring;
        bool (*notify)(struct virtqueue *vq);
    }
    struct virtqueue {
        struct virtio_device *vdev;
    }

    // inheritance:
    //   virtio_mmio_device -> virtio_device
    //   virtio_pci_device -> virtio_device

    // constructor
    vring_new_virtqueue(..., bool (*notify)(struct virtqueue *), ... )

methods of virtqueue and vring
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

virtqueue methods::

    virtqueue_add_*()
    virtqueue_get_buf()
    virtqueue_kick()
    virtqueue_disable_cb()
    virtqueue_enable_cb*()

``virtqueue_kick()`` is MMIO::

    virtqueue_kick() 
    - virtqueue_kick_prepare()
    - virtqueue_notify()
      - vring_virtqueue->notify()

    // vring_virtqueue->notify() = vm_notify() or vp_notify()
    vm_notify(struct virtqueue *vq)
      virtio_mmio_device *vm_dev = to_virtio_mmio_device(vq->vdev);
      writel(vq->index, vm_dev->base + VIRTIO_MMIO_QUEUE_NOTIFY);
        
    vp_notify(struct virtqueue *vq)
      virtio_pci_device *vp_dev = to_vp_device(vq->vdev);
      iowrite16(vq->index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NOTIFY);

virtio net
~~~~~~~~~~
::

    struct virtnet_info {
        virtio_device *vdev;
        virtqueue *cvq;
        net_device *dev;

        send_queue *sq;
        receive_queue *rq;
    }

    struct send_queue {
        struct virtqueue *vq;
        struct scatterlist sg[MAX_SKB_FRAGS + 2];
    }

    struct receive_queue {
        struct virtqueue *vq;
        struct napi_struct napi;
        struct scatterlist sg[MAX_SKB_FRAGS + 2];
    }

network driver netdev and NAPI::

    net_device_ops virtnet_netdev {
        .ndo_start_xmit      = start_xmit
        .ndo_poll_controller = virtnet_netpoll   // napi_struct
    }
    netif_napi_add(..., virtnet_poll, ...)


transmit use virtqueue_add_outbuf() and virtqueue_kick()::

    netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
    // virtnet_info *vi = netdev_priv(dev);
    // send_queue *sq = &vi->sq[qnum];
    
    - xmit_skb(struct send_queue *sq, struct sk_buff *skb)
      
      - virtqueue_add_outbuf(sq->vq, sq->sg, num_sg, skb, GFP_ATOMIC);
    
    - virtqueue_kick(sq->vq)

receive related::

    // poll() use virtqueue_get_buf()
    virtnet_poll(struct napi_struct *napi, int budget)
    // struct receive_queue *rq = container_of(napi, struct receive_queue, napi);
    
    - virtqueue_receive(rq, budget)

      - virtqueue_get_buf(rq->vq, &len);
      - receive_buf(vi, rq, buf, len);

    - napi_complete_done(napi, received);
    - napi_schedule(napi);
    - virtqueue_poll(rq->vq, r);
    - virtqueue_disable_cb(rq->vq);

    // napi_schedule
    napi_schedule() <= skb_recv_done() <= callbacks[rxq2vq(i)], virtio_device vdev->config->find_vqs(callbacks)
    napi_schedule() <= virtnet_napi_poll(), virtnet_netpoll()
    
    // napi enable/disable
    virtnet_open(), virtnet_restore(), refill_work() =>
    virtnet_napi_enable() => napi_enable() + napi_schedule()
    // CONFIG_PM_SLEEP: virtnet_freeze()/virtnet_restore()
    // INIT_DELAYED_WORK(&vi->refill, refill_work); // schedule_delayed_work()

