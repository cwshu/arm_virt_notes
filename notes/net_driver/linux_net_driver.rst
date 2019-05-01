Linux Network Driver
====================

Document
--------

- `LDD3 Ch17 Network Drivers <http://www.makelinux.net/ldd3/?u=chp-17>`_
- `2008, Understanding the Kernel Network Layer <http://stoa.usp.br/leitao/files/-1/3689/network.pdf>`_
- `Linux Foundation wiki - NAPI <https://wiki.linuxfoundation.org/networking/napi>`_

NAPI Note
---------

- NAPI receive 跟傳統 receive 的差別, 基本上還是 interrupt v.s. polling 兩種方式處理 IO 的差異.
- 傳統 receive: interrupt

  - device transfer data and send interrupt.
    kernel 在 interrupt handler 時, 從該 interrupt 對應的 device 的 rx_ring 接收資料.
  - 每個 device 可客製化 interrupt handler 來接收資料.

- NAPI receive: mixing interrupt and polling.

  - kernel 使用一個 softirq 定期去 poll 每個 device 的 rx_ring 來接收資料.
  - 每個 device 可客製化 poll function 來接收資料.
  - 該 device 的 rx_ring 要不要被 poll, 是可以被開關的: ``netif_rx_schedule/netif_rx_complete()``
  - Mixing interrupt and polling:

    當 driver 收到 interrupt 時, 在 interrupt handler 暫時 disable interrupt 加上開啟 poll mode.
    等同接下來一段時間的 packet 都共用第一個 interrupt.

    - 一段時間: 目前設定為 2 jiffies, 1 jiffie 為一次 timer interrupt 的周期 (1 ~ 10ms, based on kernel config)
    - `[Linux 4.12 patch] <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7acf8a1e8a28b3d7407a8d8061a7d0766cfac2f4>`_ 
      預設 2 ms, 可透過 ``sysctl() netdev_budget_usecs`` 調整.

  - 定期: 透過 scheduler and softirq 的機制來處理. 如果提升 ``ksoftirqd`` 的 priority 可以增加 CPU heavy loading 時的處理頻率.

- More: :ref:`NAPI`

NAPI API
--------

register poll function::

    netif_napi_add()
    netif_napi_del()

add/remove device to the ``poll_list``::

    napi_schedule() = napi_schedule_prep() + __napi_schedule()
    napi_complete()

temporarily enable/disable polling one device::

    napi_enable()
    napi_disable()


The network driver need to do 2 things to use NAPI.

- implement ``napi_struct->poll()``, register by ``netif_napi_add()``
- implement interrupt handler which disable interrupt and do ``napi_schedule()``

.. _NAPI:

NAPI more
---------

Motivation
~~~~~~~~~~

- faster NIC cause many interrupts
- Adaptive interrupt coalescing
- Prevent livelock
- More fairness in the handling devices

Intro
~~~~~

- Mixes interrupts with polling
- Process multiples frames during an interrupt
- Packet throtting: drop packets more fast when system is overwhelmed. kernel doesn't aware and NIC drop them.

Ingress workflow
~~~~~~~~~~~~~~~~

- NAPI softirq: ``net_rx_action()``

  - browses the global queue ``poll_list`` that have something in the ingress queue. 
    (``softnet_data->poll_list``, list of ``napi_struct``)
  - invoke ``napi_struct->poll()`` virtual function

    - like the interrupt handler in pre-NAPI design

  - stops when following condition is met:

    - No more devices
    - Run for too long (2 jiffies)
    - Reached a given upper bound limit (budget, return value of ``napi_struct->poll()``)
    - IRQ enabled

- ``struct napi_struct``

  - Each interrupt vector need one ``napi_struct``
  - register: ``netif_napi_add()``, ``netif_napi_del()``

    - ``netif_napi_add()`` initialize struct (set poll function) and register it to global hash table (``napi_hash``)

  - ``softnet_data->poll_list``
  
    - ``napi_schedule()``: add device to the ``poll_list``. It equals to ``napi_schedule_prep()`` + ``__napi_schedule()``
    - ``napi_complete()``: remove device from the ``poll_list``
    - ``napi_enable/disable()``: can temporarily enable/disable polling one device (``NAPI_STATE_SCHED``)

- driver need to implement

  - ``napi_struct->poll()``
  - interrupt handler which disable interrupt and do ``napi_schedule()``
 
source code::
  
    // include/linux/netdevice.h, net/core/dev.c

    open_softirq(NET_RX_SOFTIRQ, net_rx_action)
    net_rx_action() => net_poll() => napi_struct->poll()
    timer_limit = jiffies + 2 // before Linux 4.12
    
    softnet_data->poll_list // list of napi_struct
    // net_rx_action() traverse it
    // napi_schedule()/napi_complete() add to/remove from it

    napi_hash // hash table of napi_struct
    // netif_napi_add() add to it
    // napi_by_id() find napi_struct by hash id

Scheduling Issues
~~~~~~~~~~~~~~~~~

NAPI moves processing to softirq level.
This also has the effect that the priority of ksoftirq needs to be considered when running very CPU-intensive applications and networking to get the proper balance of softirq/user balance.
Increasing ksoftirq priority is to cure problems with low network performance at high CPU load.

Content
-------

`LDD3 Ch17 Network Drivers <http://www.makelinux.net/ldd3/?u=chp-17>`_

- device registeration: connecting kernel
- open & closing driver
  
  - like char/block device's ``open()`` method.
  - triggered by ``ifconfig`` command, syscall interface is ``ioctl()`` => set IF_UP flag or mac address
  - IF_UP calls open method of device
  - open method

    - ``netif_start_queue()``: start transmit queue

- packet transmittion
- packet reception
- interrupt handler

Content2
--------

- `2008, Understanding the Kernel Network Layer <http://stoa.usp.br/leitao/files/-1/3689/network.pdf>`_
- `Linux Foundation wiki - NAPI <https://wiki.linuxfoundation.org/networking/napi>`_

contents

- registeration

  - global list of network devices
  - device is not a file
  - ``register_netdev(net_device dev) => dev->init()``

- interface status

  - ``net_device->state |= __LINK_STATE_START``
  
    - set by ``dev_open()``, cleared by ``dev_close()``
    - ``__LINK_STATE_XOFF``, ``__LINK_STATE_NOCARRIER``

- Packet Reception(IRQ)

  - device receive a packet and generate a IRQ.
  - function from interrupt vector table is called. (e.g. ``e1000_intr()``)
  - the driver copies the frame into an input queue.
  - Then, the kernel pass the frame to the upper layer (e.g. IP)
  - an skb is allocated using ``dev_alloc_skb()`` before DMA accomplishment and after the packet reception.
  - call ``netif_rx()``, ``netif_receive_skb()``, then ``ptype->prev()``

Additional Questions
--------------------

- In Linux Kernel, what is a difference between kernel thread and softirq?

  - ksoftirqd is a kernel thread.
  - softirq <https://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-9.html#softirqs>
