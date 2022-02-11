VFIO details
============

Example Usage
-------------

`example in vfio-pci <http://elixir.free-electrons.com/linux/v4.13/source/Documentation/vfio.txt#L118>`_

- prepare

  - unlink device to pci driver in sysfs
  - link device to vfio driver in sysfs (and bind to vfio group)
  - provide the user with access to the vfio group if unprivileged operation is desired

- user access vfio

  - Open vfio container (``/dev/vfio/vfio``) and vfio group (e.g. ``/dev/vfio/26``)
  - Add group to container: ``ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);``
  - Set IOMMU model of the container: ``ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU);``
  - Use ``mmap()`` to allocate some space and setup a DMA mapping: ``ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);``
  - Get device from vfio_group: ``device = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:06:0d.0");``
  - Get device information for userspace driver?

    - ``device_info``: ``VFIO_DEVICE_GET_INFO``
    - ``region_info``: ``VFIO_DEVICE_GET_REGION_INFO``
    - ``irq_info``: ``VFIO_DEVICE_GET_IRQ_INFO``

- user access vfio2

  - vfio container, group, device are all fd.
  - dma_map: ``struct vfio_iommu_type1_dma_map``
  - ``vfio_device_info``, ``vfio_region_info``, ``vfio_irq_info``

- VFIO User API: ``include/linux/vfio.h``
- `VFIO bus driver API <http://elixir.free-electrons.com/linux/v4.13/source/Documentation/vfio.txt#L247>`_

  - ``vfio_add_group_dev()``: vfio core begin tracking the specified iommu_group, register the specified dev by VFIO bus driver.

prepare for vfio-platform
~~~~~~~~~~~~~~~~~~~~~~~~~

::

    echo "vfio-platform" > /sys/bus/$bus/devices/$device/driver_override
    echo $device > /sys/bus/$bus/devices/$device/driver/unbind
    echo $device > /sys/bus/$bus/drivers_probe

    1. set "vfio-platform" to device/driver_override
    2. set device_name to driver/unbind
    3. set device_name to bus/drivers_probe

``device/driver_override``

    規定只有跟 ``driver_override`` 的 value 同名的 driver 才能 binding 到 device.
    當 binding 的方式為 ``bus/drivers_probe`` 時適用. 其他情況不確定.

    ::

        DEVICE_ATTR_RW(driver_override);
        // => sysfs device_attribute
        //    setter, getter = driver_override_store, driver_override_show

        // driver_override_store() set parameter to [platform_device] pdev.driver_override

        // platform_match() use pdev.driver_override
        // of, acpi, id table match?

``driver/unbind``

    把 driver 跟現在 driver binding 的 device 進行 unbind.

    ``bus_find_device_by_name(driver->bus, buf)`` + ``device_release_driver(dev)``

``bus/drivers_probe``

    drivers_probe 會去掃描 bus 下的所有 driver.
    如果這些 driver 跟 device 有 match 的話(``platform_match()``), 就讓 driver 跟 device 進行 binding(probe).

    ::

        BUS_ATTR(drivers_probe, S_IWUSR, NULL, store_drivers_probe);

        store_drivers_probe() => bus_rescan_devices_helper() => device_attach()

        // device_attach() walks through the list of driver in bus.
        // calls driver_probe_device() for each pair if driver_match_device()
           // for each driver in bus: __device_attach_driver()
           __device_attach_driver()
               if driver_match_device() // use pdev.driver_override.
                   driver_probe_device()

        driver_match_device() => driver->bus->match() => platform_match() for platform_device

background API:

1. device, driver, and bus use DEVICE_ATTR, DRIVER_ATTR, BUS_ATTR to declare sysfs entry

   1.a. store and show operations are like setter and getter.

一些整理?
~~~~~~~~~

- ``vfio_device_fops`` 裏面的 ``region_info``, ``irq_info`` 很像 Linux Driver 使用硬體的 API: ``request_memory_region()``, ``request_irq()``.

- set vfio_group to vfio_container (ioctl vfio_group) => the remaining ioctl becomes available

  - VFIO IOMMU access
  - get device fd in vfio_group

- vfio device API

  - describe device
  - IO region
  - read/write/mmap offset
  - mechanism to register interrupt notification

kernel 內部實作
---------------

VFIO 的介面有三種 fd: container, group, and device.
固對應的 Linux Kernel 也有實作三組 fops 介面: ``vfio_fops``, ``vfio_group_fops``, ``vfio_device_fops``.

kernel 內部元件跟介面的關聯, 可以參考別人整理的一張圖.

.. image:: https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/resource/BLOGS_UPLOADED_IMAGES/vfio_fig3.png

VFIO 內部由三個部份所組成: VFIO core + VFIO bus driver + VFIO IOMMU driver.
MMIO region map 疑似在 VFIO bus driver 完成的

details
~~~~~~~

- vfio_fops:

  - open, release, read, write
  - read/write: iommu_driver->ops->read/write
  - ioctl:

    - get_api_version, check_extension
    - set_iommu

- vfio_group_fops:

  - open, release, ioctl
  - ioctl: get_status, set/unset_container, get_device_fd

- vfio_device_fops

  - abstraction of device->ops

介面 detail
~~~~~~~~~~~

- fops

  - ``vfio_fops`` 是基於 misc_device: ``misc_register(&vfio_dev);``
  - ``vfio_group_fops`` 是基於 char device: ``cdev_init(&vfio.group_cdev, &vfio_group_fops);``
  - ``vfio_device_fops`` 是基於 anonymous fd, 當 process 呼叫 ``vfio_group_get_device_fd()`` API 時會動態產生該介面的 fd 給 process.

- ``vfio_device_fops`` 的 functions, 會去呼叫參數 vfio_device 裏面的 op structure.

  - 比如說 ``vfio_device_read()`` 會呼叫 ``vfio_device->ops->read()``

- ``vfio_fops`` 的 ioctl() 除了三個 option 以外, 都會 redirect 到 ``vfio_iommu_driver.ioctl()``

內部元件 detail
~~~~~~~~~~~~~~~

- ``vfio_group_set_container(struct vfio_group *group, int container_fd)``

  - [iommu_driver] driver->ops->attach_group(container->iommu_data, group->iommu_group);

- vfio_iommu_driver 實作了 container 的 ioctl options VFIO_IOMMU_MAP_DMA

  - container fops ioctl() VFIO_IOMMU_MAP_DMA 會 redirect 到 ``vfio_iommu_driver.ioctl()``

- vfio_pci (vfio bus driver) 如何跟 vfio interface connect?

  - vfio_pci 會 implement 一個 pci driver 並註冊: ``pci_register_driver()``
  - vfio_pci 註冊後, 會創造對應的 ``vfio_device`` 跟 ``vfio_group``, 並且用一個 idr 去 index 該 ``vfio_group`` pointer.
  - vfio_group fops open() 時, 會透過 idr 找到 vfio_pci 產生的 ``vfio_group``

other about functionality
-------------------------

- 將 DMA 暴露到 userspace
- 將 interrupt 暴露到 userspace

  - userspace 使用 eventfd
  - 透過 VFIO_DEVICE_GET_REGION_INFO 得到 interrupt 訊息
  - 透過 ``eventfd()`` 產生一個 eventfd, 並用 VFIO_DEVICE_SET_IRQ 把這個 eventfd 跟 interrupt 相連起來.
  - 可以 ``select/poll/epoll()`` 該 eventfd

QEMU VFIO (VFIO_PLATFORM)
-------------------------

.. _qemu_vfio_trace:

User Interface
~~~~~~~~~~~~~~

QEMU device parameter::

  1. -device vfio-platform,host=ee300000.sata
  2. -device vfio-platform,sysfsdev=/sys/bus/platform/devices/ee300000.sata

Source Code::

  hw/vfio/platform.c

Ref: https://elinux.org/R-Car/Virtualization/VFIO#SATA_Pass-Through


Kernel APIs
~~~~~~~~~~~

VFIO:

  1. container fd::

       - from /dev/vfio/vfio
       - ioctl()
         - VFIO_IOMMU_MAP_DMA
         - VFIO_IOMMU_UNMAP_DMA
         - VFIO_SET_IOMMU

  2. group fd::

       - from /dev/vfio/$GROUP
       - ioctl()
         - VFIO_GROUP_SET_CONTAINER
         - VFIO_GROUP_GET_DEVICE_FD
         - VFIO_GROUP_GET_STATUS, VFIO_GROUP_FLAGS_VIABLE

  3. device fd::

       - from VFIO_GROUP_GET_DEVICE_FD
       - ioctl()
         - VFIO_DEVICE_GET_REGION_INFO
         - VFIO_DEVICE_GET_IRQ_INFO
         - VFIO_DEVICE_SET_IRQS

       - mmap()
       - read/write()

KVM:

  1. VM fd::

       - from KVM_CREATE_VM
       - ioctl()
         - KVM_CREATE_DEVICE
         - KVM_IRQFD

  2. vfio_kvm_device_fd::

       - from KVM_CREATE_DEVICE, type=KVM_DEV_TYPE_VFIO
       - ioctl()
         - KVM_SET_DEVICE_ATTR

  (p.s. /dev/kvm and VCPU fd are not used here)

Trace code
~~~~~~~~~~

Data::

  FDs:
    - vbasedev->fd
    - vfio_kvm_device_fd

  QEMU MemoryRegion:
    - region->mem
    - region->mmaps[i].mem

  QEMU EventNotifier:
    - intp->interrupt
    - intp->unmask

  QEMU EventNotifier
    // Abstraction of eventfd, contains rfd and wfd
    // APIs:
    //   event_notifier_set(): set wfd
    //   event_notifier_test_and_clear(): check if rfd has been triggered
    //   event_notifier_get_fd(): return rfd (used by epoll())

Code::

  [MMIO]
  vfio_populate_device()
    vfio_region_setup() // configure vdev->regions[i]
      ioctl: VFIO_DEVICE_GET_REGION_INFO
      memory_region_init_io(region->mem, &vfio_region_ops)
      // configure region->nr_mmaps and region->mmaps
      //   1. region->nr_mmaps = 1
      //   2. vfio_setup_region_sparse_mmaps()

  /*
   * vfio_region_ops seems like a fallback mechanism of MMIO,
   * if some MMIO registers wouldn't be covered by region->mmaps.
   * (but why and which condition the MMIO register isn't covered by region->mmaps?)
   *
   * Guest OS access of MMIO register will trap and the switch to QEMU,
   * then QEMU could access MMIO register via read/write() to VFIO device fd.
   * (QEMU is more like a userspace driver using VFIO in this fallback mechanism.)
   */
  vfio_region_ops
    vfio_region_read()
      pread(vbasedev->fd, region->fd_offset + addr)
      vbasedev->ops->vfio_eoi(vbasedev);

    vfio_region_write()
      pwrite(vbasedev->fd, region->fd_offset + addr)
      vbasedev->ops->vfio_eoi(vbasedev);

  /*
   * Create GPA to HPA mapping of MMIO registers
   */
  vfio_platform_realize()
    for each vdev->regions:
      vfio_region_mmap(vdev->regions[i])
        for each mmap (nr_mmaps):
          // Use VFIO API to create mapping of MMIO register (VA => PA).
          // QEMU process could see the MMIO register.
          region->mmaps[i].mmap = mmap(region->vbasedev->fd)

          // Use KVM API to create mapping of MMIO register (GPA => HPA).
          // KVM API argument of MMIO register is VA in QEMU, so QEMU needs to see MMIO registers at first.
          // Guest OS could see MMIO registers.
          memory_region_init_ram_device_ptr(&region->mmaps[i].mem, region->mmaps[i].mmap)
          memory_region_add_subregion(region->mem, region->mmaps[i].mem)

  [IRQ]
  vfio_populate_device()
    ioctl: VFIO_DEVICE_GET_IRQ_INFO
    vfio_init_intp()
      event_notifier_init(intp->interrupt)
      if VFIO_IRQ_INFO_AUTOMASKED:
        event_notifier_init(intp->unmask)

  sysbus_connect_irq()
    sbd->connect_irq_notifier()
      vfio_start_irqfd_injection()
        vfio_set_trigger_eventfd()
          kvm_vm ioctl: KVM_IRQFD

          ioctl: VFIO_DEVICE_SET_IRQS
            .flag = VFIO_IRQ_SET_DATA_EVENTFD | VFIO_IRQ_SET_ACTION_TRIGGER
            .data = fd // from intp->interrupt

  [IOMMU]
  vfio_memory_listener
    vfio_listener_region_add()
      vfio_dma_map() => ioctl: VFIO_IOMMU_MAP_DMA

    vfio_listener_region_del()
      vfio_dma_unmap() => ioctl: VFIO_IOMMU_UNMAP_DMA

  [Init]
  vfio_base_device_init()
    vfio_get_group(groupid)
      if new VFIO group:
        open() the VFIO group (/dev/vfio/$GROUP)
          ioctl: VFIO_GROUP_GET_STATUS, VFIO_GROUP_FLAGS_VIABLE

        vfio_connect_container()
          // see below

        qemu_register_reset(vfio_reset_handler)
          vfio_reset_handler => VFIODeviceOps: vfio_compute_needs_reset, vfio_hot_reset_multi

    vfio_get_device()
      ioctl: VFIO_GROUP_GET_DEVICE_FD
      ioctl: VFIO_DEVICE_GET_INFO

    vfio_populate_device()
      ioctl: VFIO_DEVICE_GET_REGION_INFO
      memory_region_init_io()

      ioctl: VFIO_DEVICE_GET_IRQ_INFO
      vfio_init_intp()

  vfio_connect_container()
    if existing VFIO container:
      ioctl: VFIO_GROUP_SET_CONTAINER (group fd -> container fd)
      vfio_kvm_device_add_group()
        kvm_vm ioctl: KVM_CREATE_DEVICE, type=KVM_DEV_TYPE_VFIO
        vfio_kvm_device_fd ioctl: KVM_SET_DEVICE_ATTR, attr=KVM_DEV_VFIO_GROUP_ADD

    if new VFIO container:
      open() the VFIO container (/dev/vfio/vfio)
      vfio_init_container()
        ioctl: VFIO_GROUP_SET_CONTAINER
        ioctl: VFIO_SET_IOMMU: iommu_type
      vfio_ram_block_discard_disable()
      vfio_host_win_add(container, iova_pgsizes) // VFIOHostDMAWindow
      vfio_kvm_device_add_group()
      register vfio_memory_listener
