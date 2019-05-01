KVMMemoryListener
=================

Interface
---------

- MemoryRegionSection: 代表一段 memory address range; memory 特性跟指向的 MemoryRegion 一致

  - memory address (``MemoryRegionSection* section``)

    - gpa start: (hwaddr) section->offset_within_address_space
    - hva start:  (void*) memory_region_get_ram_ptr(section->mr) + section->offset_within_region
    - size: section->size

- MemoryListener API::

    MemoryListener->region_add(MemoryListener *listener, MemoryRegionSection *section)
    MemoryListener->region_del(MemoryListener *listener, MemoryRegionSection *section)

    把 section 加入到 guest memory / 從 guest memory 刪除.
    以 KVMMemoryListener 為例, 就是透過 KVM API 對 guest OS 加入/刪除 memory page.

  - KVMMemoryListener 實作 ``kvm_region_add() / kvm_region_del()``
  - KVM API: ``kvm_vm_ioctl(KVM_SET_USER_MEMORY_REGION);``

Contents
--------

- ``kvm_region_add() / kvm_region_del()`` 都會轉成 ``kvm_set_phys_mem()``, 用 bool 參數 add 來分辨.
- ``kvm_set_phys_mem()``

  - 功能: 把 MemoryRegionSection 這段 hwaddr range 加入/刪除到 KVM 的記憶體管理.
  - 前置行為

    - 把 address range 進行 page alignment, 只留中間有 page-aligned 的 memory (小於 page size 則直接消失)
    - 如果是非 ram 的 MemoryRegion, 則大部份 MR 會直接取消 mapping 或是從 region_add 轉成 region_del. 具體 condition 請見 source code.

  - 實作方式:
  
    - 先當成 region_del, 把現有的 KVMSlot 之中, 跟 MemoryRegionSection 有交集的全部取出, 並且砍掉這些交集.
    - 如果是 region_add 的話, 再把該 MemoryRegionSection 轉成 KVMSlot 之後, 加入 KVM 的記憶體管理.
    - 例外: 如果是 region_add, 並且 Section 的 address range 包含於取出的 KVMSlot, 就只更新 KVMSlot 的 flag. 並且函式直接 return
    - 砍掉交集的演算法: ``KVMSlot old - MemoryRegionSection new``
  
      - 移除 old 的 KVMSlot
      - 新增兩個 KVMSlot prefix, suffix = ([old.start, new.start], [new.end, old.end])
  
  - API
  
    - 取出交集的 KVMSlot: ``kvm_lookup_overlapping_slot()``
    - 加入 KVM 的記憶體管理 / 砍掉交集: ``kvm_set_user_memory_region()``
    - 更新 KVMSlot 的 flag: ``kvm_slot_update_flags()`` => ``kvm_set_user_memory_region()``
  
  - API detail::
  
      static KVMSlot *kvm_lookup_overlapping_slot: 找出跟 (start_addr, end_addr) overlap 且 start_addr 最小的 KVMSlot, KVMSlot from kml
          - overlap algor: a->end > b->start && a->start < b->end
      kvm_slot_update_flags(kml, mem, mr): 從 mr(MemoryRegion) 抽出 kvm_mem_slot 的 flag, 如果需要更新則透過 kvm_set_user_memory_region() 更新.

- KVM Memory API wrapper: ``kvm_set_user_memory_region(KVMMemoryListener *kml, KVMSlot *slot)``

  - call ``kvm_vm_ioctl(KVM_SET_USER_MEMORY_REGION);``
  - ``struct KVMSlot`` to ``struct kvm_userspace_memory_region``::

      int        slot | (kml->as_id << 16) => slot
      hwaddr     start_addr                => guest_phys_addr (gpa)
      void*      ram                       => userspace_addr  (uaddr)
      ram_addr_t memory_size               => memory_size
      int        flags                     => flags

- ``KVMSlot``

  - stored in KML: `KVMMemoryListener kml; KVMSlot *mem = &kml->slots[i];``
  - 數目(nr_slots init): ``s->nr_slots = kvm_check_extension(s, KVM_CAP_NR_MEMSLOTS);``

More

- 開始有 pointer 是 hva, hwaddr 是 gpa 的感覺了.
- function path: ``kvm_region_add()`` => ``kvm_set_phy_ram()`` => ``kvm_set_user_memory_region()``
- data path: ``MemoryRegionSection`` => ``KVMSlot`` => ``struct kvm_userspace_memory_region``

KVM memory API
--------------
- VM ioctl - KVM_SET_USER_MEMORY_REGION

  - `API doc <http://elixir.free-electrons.com/linux/v4.11/source/Documentation/virtual/kvm/api.txt#L930>`_
  - create, modify, and delete guest physical memory slot. it controls guest physical memory.
  
    - slot can be moved, or its flag can be modified. it can't be resized.
    - slots may not overlapped in GPA space.

  - mapping the memory page in HVA to GPA (addrs trans: GPA => HPA)
  - parameter: struct kvm_userspace_memory_region

    - ``guest_phys_addr`` is GPA and ``userspace_addr`` is HVA
    - slot: multi address space
    - recommand last 21 bits of gpa & uaddr is identical (large page )

  - example

    - https://senselab.tw/git/u1240976/arm_virt_test_code/blob/master/memory_map_test/qemu_patch/host_map.diff#L32

      - mapping: GPA = 0x80000000, HVA = ``mmap()`` return value.
      - fails currently.

Misc
----

KVM memory
~~~~~~~~~~

- mmap VM fd is gpa ??
- kvm_vm_ioctl_set_memory_region() => __kvm_set_memory_region()

  - kvm_memslots, kvm_memory_slot, slot = (as_id, id)
  - [arm] arch_prepare_memory_region() => unmap_stage2_range(kvm, mem->guest_phys_addr, mem->memory_size);
  
