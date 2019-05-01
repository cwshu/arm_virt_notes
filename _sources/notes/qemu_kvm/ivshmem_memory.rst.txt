Memory Usage in ivshmem
=======================

ivshmem 處理 shared memory 的原理, 簡單來說就是.

- ivshmem 建立了一個 RAM type 的 ``MemoryRegion`` 來封裝 host OS 上的 shared memory.
- ivshmem 將該 ``MemoryRegion`` 註冊成自己 PCI device 的 BAR2. 
  QEMU PCI framework 會將所有 PCI device 的 BAR 加入到 ``system_memory`` 當中, 進行 guest OS memory 的管理.
- 當 ``MemoryRegion`` 被加入到 ``system_memory`` 時, QEMU 就會透過 KVM API 對 guest OS 的記憶體進行對應的操作.
  舉例來說操作可能是在 page table 建立 memory mapping 或是為了 trap-and-emulate 而把 page table 的 mapping 刪掉, 這根據 ``MemoryRegion`` 的 type 決定.
  ivshmem 這邊的操作則是將 shared memory mapping 到 guest OS 的記憶體.

  - 細節可參考: :ref:`MR_transaction`

接下來就個別講解細節.

ivshmem CLI option
------------------
ivshmem CLI option 包含 shared memory 的路徑跟大小, 這兩個資料會被傳入 IVShmemState 的 class member 當中.

- CLI option: ``-device ivshmem,shm=hello,size=1`` => ``IVShmemState``
  ::
    
      IVShmemState->shmobj = "hello" // shm
      IVShmemState->sizearg = 1      // size

- 細節請參考: :ref:`qdev_property`

desugar_shm(): create ``MemoryRegion``
--------------------------------------

Create ``TYPE_MEMORY_BACKEND_FILE`` and ``MemoryRegion`` by ``IVShmemState->shmobj``.
Assign the ``MemoryRegion`` to ``IVShmemState->hostmem``.

- ``hostmem = object_new(TYPE_MEMORY_BACKEND_FILE)``
- ``hostmem->properties["mem-path"] = "/dev/shm/" + IVShmemState->shmobj``
- ``user_creatable_complete(hostmem)``

  a. create ``MemoryRegion``
  b. ``mmap()`` file to QEMU process memory

register ``MemoryRegion`` as PCI BAR2
-------------------------------------

ivshmem 將該 ``MemoryRegion`` 註冊成自己 PCI device 的 BAR2.

- ``IVShmemState->ivshmem_bar2 = host_memory_backend_get_memory(IVShmemState->hostmem)``
- ``pci_register_bar(PCI_DEVICE(s), 2, attr, IVShmemState->ivshmem_bar2);``

Then, we should dig into QEMU PCI framework.

PCI framework
-------------

QEMU PCI framework 會將所有 PCI device 的 BAR 加入到 ``system_memory`` 當中, 進行 guest OS memory 的管理.

- pci_register_bar(): 把 ``MemoryRegion`` 的資料放到 pci_dev->io_region[bar_num]
- pci_update_mapping(): 

  - 更新 pci_dev->io_region[bar_num], 將該 ``MemoryRegion`` 加入 ``system_memory`` 當中 (``memory_region_add_subregion()``)
  - memory_region_add_subregion_overlap(r->address_space, r->addr, r->memory, 1); // PCIIORegion* r;
  - PCI device 使用的 address_space 為何?

    - ``PCIIORegion->address_space`` is from ``PCIBus->address_space_{mem,io}``
    - ``PCIBus->address_space_mem`` is init at PCI Host Bridge start

  - More: :ref:`MR_transaction`, :doc:`/notes/qemu_kvm/qemu_memory_kml`

Details
-------

1. TYPE_MEMORY_BACKEND_FILE::

     // user_creatable_complete() create MemoryRegion and do mmap()
     user_creatable_complete() => host_memory_backend_memory_complete() => file_backend_memory_alloc() => memory_region_init_ram_from_file()
     
     // inheritance: TYPE_MEMORY_BACKEND_FILE => TYPE_MEMORY_BACKEND => TYPE_OBJECT
     [TYPE_MEMORY_BACKEND] UserCreatableClass* ucc->complete = host_memory_backend_memory_complete
     [TYPE_MEMORY_BACKEND_FILE] HostMemoryBackendClass* bc->alloc = file_backend_memory_alloc 

     host_memory_backend_memory_complete()
     => bc->alloc() = file_backend_memory_alloc() 
     => memory_region_init_ram_from_file()

2. pci_{set,get}_{byte,word,...}: pointer dereference with endianness.

struct

- PCIDevice
  
  - PCIIORegion io_regions[]; // all PCI BAR, each element of array is single PCI BAR
  - PCIBus bus;

  - uint8_t* config; // PCI config space. each PCI BAR has some space.
  - uint8_t* wmask, cmask;

- PCIIORegion

  - pcibus_t addr, size; // MMIO region, addr may equal to PCI_BAR_UNMAPPED;
  - MemoryRegion memory; // one PCI BAR maps to one QEMU MemoryRegion
  - MemoryRegion address_space; // one of pci_dev->bus->address_space_{mem,io}

- PCIBus

  - MemoryRegion address_space_io, address_space_mem;
  - init at PCI Host Bridge (e.g. ``i440fx_init()``)

misc: gdb print MemoryRegion linked list::

    (gdb) memory_region_p mr->subregions->tqh_first                          
    MemoryRegion 'kvm-apic-msi': size=0/100000, is_ram 0, is_ro 0, hwaddr fee00000, callback addr 0x555555eeae00
    (gdb) memory_region_p mr->subregions->tqh_first->subregions_link.tqe_next
    MemoryRegion 'ram-above-4g': size=0/40000000, is_ram 0, is_ro 0, hwaddr 100000000, callback addr 0x555555ee6780
    (gdb) memory_region_p mr->subregions->tqh_first->subregions_link.tqe_next->subregions_link.tqe_next
    MemoryRegion 'ram-below-4g': size=0/c0000000, is_ram 0, is_ro 0, hwaddr 0, callback addr 0x555555ee6780
    (gdb) memory_region_p mr->subregions->tqh_first->subregions_link.tqe_next->subregions_link.tqe_next->subregions_link.tqe_next
    MemoryRegion 'pci': size=1/0, is_ram 0, is_ro 0, hwaddr 0, callback addr 0x555555ee6780
