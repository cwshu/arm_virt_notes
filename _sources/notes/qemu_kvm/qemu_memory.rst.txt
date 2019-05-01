QEMU Memory Management
======================

MMIO exit handling
------------------

- KVM exit: QEMU get return from KVM ``ioctl()`` syscall, and use ``address_space_rw()`` to handle MMIO exit.
  [at ``kvm_cpu_exec() in ./kvm-all.c``.]
- ``address_space_rw()``: ``AddressSpace`` dispatch MMIO exit to specific ``MemoryRegion`` based on ``hwaddr``

  - Control flow (MMIO write)::
  
      (1) address_space_rw() => address_space_write() => address_space_write_continue() =>
      (2) memory_region_dispatch_write() => access_with_adjusted_size() => memory_region_write_accessor() =>
      (3) MemoryRegion mr; mr->ops->write( ... )

  - how to find specific ``MemoryRegion`` by guest addr: ``address_space_translate()``
  - source: ``memory.c``, ``exec.c``

- example for device MemoryRegion->write() [not finish]
  
  - ivshmem_bar0

.. _MR_transaction:

MemoryListener and MemoryRegion transaction
-------------------------------------------

當 ``MemoryRegion`` 被加入或移出 ``MemoryRegion`` tree 時. QEMU 會透過 ``MemoryListener`` 去更新 guest OS 的 memory. 這個更新過程叫作 MemoryRegion transaction.

- 簡稱

  - ``MemoryRegion``: MR
  - ``MemoryRegionSection``: MRS

- MemoryListener API: 

  - 每次 memory 的更新範圍為一組 ``(start, len)``, 包裝成 MRS 的型態.
    region_add()/region_del() 分別為新增(更新)跟刪除這段 memory.
  - API: ``region_add(MemoryListener *listener, MemoryRegionSection *section)``
  - 舉例來說, :doc:`/notes/qemu_kvm/qemu_memory_kml` 就會透過 KVM API 去更新 guest OS 的 memory.

- MemoryRegion transaction: ``memory_region_transaction_commit()``

  - 當 MR tree 新增/刪除 node 時, 會 commit 一個 MR transaction, 來更新 MR tree 對應到的 ``AddressSpace``.
  - MR transaction 更新方式: 讓該 ``AddressSpace`` 產生新的 ``FlatView``, 
    並跟上次 commit 的 ``FlatView`` 比較 memory 的差異,
    把這些 memory 差異 (包裝成 MRS 型態), 透過 MemoryListener 更新到 guest OS 的 memory 上.
  - MR transaction 會更新每個 ``AddressSpace``.
  - MR transaction 會呼叫 ``AddressSpace`` 之中每個 ``MemoryListener`` 各自更新一次.

- ``memory_region_transaction_commit()`` source analysis.::

    memory_region_transaction_commit()
      MEMORY_LISTENER_CALL_GLOBAL(begin, Forward);
      foreach(as: address_spaces): address_space_update_topology(as);
      MEMORY_LISTENER_CALL_GLOBAL(commit, Forward);
    
    address_space_update_topology(AddressSpace as)
      # old FlatView (previous rendering version)
      FlatView old_view = as->current_map;
      # rendering new FlatView - render_memory_region()
      FlatView new_view = generate_memory_topology(as->root); 

      address_space_update_topology_pass(as, old_view, new_view, false); // adding=false
      address_space_update_topology_pass(as, old_view, new_view, true);  // adding=true
    
    address_space_update_topology_pass()
      // update topology:
      //   delete (old_view - new_view) 
      //   update (intersect(old_view, new_view)) 
      //   add    (new_view - old_view) 
      // FlatView fr; (FlatView has fr->nr FlatRange. The ith FlatRange is accessed by fr->ranges[i])

      while(iold < old_view->nr || inew < new_view->nr)
        1. if(!adding) MEMORY_LISTENER_UPDATE_REGION(frold, as, Reverse, region_del); // iold++;
        2. if(adding)  MEMORY_LISTENER_UPDATE_REGION(frnew, as, Forward, region_nop); // inew++; iold++;
        3. if(adding)  MEMORY_LISTENER_UPDATE_REGION(frnew, as, Forward, region_add); // inew++;
    
    MEMORY_LISTENER_UPDATE_REGION()
      MEMORY_LISTENER_UPDATE_REGION(..., region_add) calls MemoryListener->region_add
      MEMORY_LISTENER_UPDATE_REGION(..., region_del) calls MemoryListener->region_del

misc
~~~~
- global memory listeners

  - a global list: ``memory_listeners``
  - call by ``MEMORY_LISTENER_CALL_GLOBAL()``

Note of global variable
-----------------------

- global variable ``address_space_memory`` is guest physical address space,
  ``system_memory`` is root MemoryRegion of ``address_space_memory``

::

    1. static MemoryRegion *system_memory;       // root MemoryRegion of address_space_memory
    2. AddressSpace address_space_memory;        // default memory space
       extern AddressSpace address_space_memory; // in include/exec/address-spaces.h

    3. static QTAILQ_HEAD(, AddressSpace) address_spaces = QTAILQ_HEAD_INITIALIZER(address_spaces);

More
----

MMIO exit
~~~~~~~~~

- emulation of trap: 

  - 2 cases 

    1. ram case - use ``memcpy()`` to emulate::

         ptr = qemu_map_ram_ptr(mr->ram_block, addr1); // get RAM backend from MR's RAMBlock
         memcpy(ptr, buf, l);
         invalidate_and_set_dirty(mr, addr1, l);

         # ram case?
         # memory_access_is_direct(mr, true)
         #   in write case: mr->ram && !mr->readonly && !mr->ram_device
         #   in read case: (mr->rom_device || mr->romd_mode) || (mr->ram && !mr->ram_device)

    2. other case - ``memory_region_dispatch_write()``

  - at ``address_space_write_continue()``
  - 3 kind of accessor: mmio, mmio_with_attrs, oldmmio
  - wrappers of mr->op->write(), write_with_attrs() ...

- ``address_space_translate()``

  - ``struct AddressSpace, AddressSpaceDispatch, MemoryRegion;``
  - address_space_translate() => address_space_translate_internal() => address_space_lookup_region() => phys_page_find()
  - ``AddressSpaceDispatch`` and ``PhysPageMap``

- ``AddressSpaceDispatch``

  - PhysPageMap, PhysPageEntry, Node(PhysPageEntry[1024])
  - PhysPageEntry: skip, ptr
  - PhysPageMap: MemoryRegionSection*, Node*

  - 4 layer page table in PhysPageMap, each entry is PhysPageEntry ??
  - PhysPageEntry array is indexed by hwaddr

  - PhysPageEntry.ptr 跟 PhysPageMap.section[section_nb] 裡的 section_nb 有關

other
~~~~~

- RAMBlock & RAMList

  - ram_addr_t is a namespace different to GPA. it depends on the order RAMBlock is created. (index RAMBlock)
  - global list RAMList ram_list is hold all RAMBlock and dirty memory bitmaps. 
  - RAMBlock
  
    - anonymous mmap and file-backed mmap.
    - file-backed mmap can be used for hugetlbfs or shared memory between host/guest.
    - qemu_ram_alloc*()
  
    - qemu_ram_set_idstr() // set, get, unset
    - qemu_get_ram_block() // ram_addr_t => RAMBlock*
    - qemu_ram_resize()
    - ram_block_add()

- ``[MemoryRegion]`` some callers of ``memory_region_init()`` 

  - vga_init() => memory_region_add_subregion_overlap() => memory_region_transaction_commit() => KVM API update guest address.
  - [?] pci_host_config_write_common() => e1000_write_config() => pci_default_write_config() => pci_update_mapping() / memory_region_set_enabled() => KVM API ...
  - vga_update_memory_access() => memory_region_add_subregion_overlap() => ...

QEMU memory doc 
---------------
- `Stefan Hajnoczi - QEMU Internals: How guest physical RAM works <http://blog.vmsplice.net/2016/01/qemu-internals-how-guest-physical-ram.html>`_

  - `中譯 <https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/20160921?lang=en>`_
  - AddressSpace/MemoryRegion

    - memory space and I/O space are both represented by AddressSpace
    - AddressSpace containes tree of MemoryRegion
    - MemoryRegion: link between guest physical address space (``hwaddr``) and the ``RAMBlocks`` containing the memory.
    - MemoryRegion also represents I/O memory, which containes callback(read/write), that is invoked on access.

  - RAMBlocks and ram_addr_t address space
    
    - each RAMBlock has pointer to ``mmap()`` memory and ram_addr_t offset. (``mmap()`` by ``qemu_ram_alloc()``)
    - global variable RAMList ram_list. it holds all RAMBlocks and dirty memory bitmaps

  - hotpluggable guest memory: pc-dimm [1] device
  - memory-backend device
  - dirty memory tracking (bitmaps in RAMList): used by live migration, QEMU TCG handling SMC code ... etc.
  - [1] DIMM means Dual In-Line Memory Module, is used in system memory in our PC.

- `QEMU docs/memory.txt <https://lxr.missinglinkelectronics.com/qemu+v2.8.0/docs/memory.txt>`_

  - MemoryRegion types: RAM, MMIO, ROM, ROM device, IOMMU region, container, alias, reservation region.
  - MemoryRegion lifecycle: 
    
    - ``memory_region_init*()``
    - ``memory_region_{add,del}_subregion()``
    - ``object_unparent()``, region attributes

  - Overlapping regions and priority
  - Visibility of MemoryRegion
  - [MMIO Operations] various contraint can be supplied to control how read()/write() callback is called

- `QEMU 对虚机的地址空间管理 <http://www.cnblogs.com/wuchanming/p/4732604.html>`_

  - pci_update_mapping()
  - qemu_mutex_lock_iothread()
  - memory_region_transaction_{begin,commit}()

- `QEMU Memory <http://people.cs.nctu.edu.tw/~chenwj/dokuwiki/doku.php?id=qemu#memory>`_ 

  - RAMBlock and RAMList
  - PhysPageDesc

- `QEMU SoftMMU <http://people.cs.nctu.edu.tw/~chenwj/dokuwiki/doku.php?id=qemu#software_mmu>`_
- [example] http://blog.csdn.net/dashulu/article/details/17090293
