DMA Mapping
===========

- [ELC2014] `Mastering the DMA and IOMMU API <http://elinux.org/images/4/49/20140429-dma.pdf>`_ 

Source Code
-----------

- headers::

    // struct dma_map_ops 
    include/linux/dma-mapping.h

    // dma_map_*(), dma_alloc_*() APIs
    include/asm-generic/dma-mapping-common.h 

    // get_dma_ops(), some of dma_*() APIs
    arch/x86/include/asm/dma-mapping.h
    arch/arm/include/asm/dma-mapping.h
    arch/arm64/include/asm/dma-mapping.h

- sources::

    // struct dma_map_ops implementations
    arch/arm/mm/dma-mapping.c: implement 4 type of dma_map_ops
    arch/arm64/mm/dma-mapping.c: implement 2 type of dma_map_ops

Intro
-----

1. DMA mapping API abstraction

   - DMA mapping ops: ``struct dma_map_ops``
   - DMA mapping API: ``dma_map_*()``, ``dma_alloc_*()`` ..., like ``dma_alloc_coherent(struct device*)``
   - DMA mapping API is the abstraction of DMA mapping ops.
     It use ``get_dma_op()`` to find correct DMA mapping ops implementation.

2. DMA mapping ops has 6 kinds of APIs.

3. ``alloc()`` op do memory allocation?

   - ``arm_dma_ops.alloc()`` can use 4 different allocators, including CMA allocator.
   - simple_allocator's ``alloc()`` use memory subsystem API ``alloc_page()``
   - ``iommu_ops.alloc()`` use gen_pool allocator

4. ``map_*()`` op do cache invalidate and clean operations

DMA mapping API
---------------

DMA mapping API/ops

- DMA mapping ops 有 6 種 operations: alloc, free, mmap, get_sgtable, map, sync.
  
  - 其中 map 跟 sync 的 API 比較多樣, 根據 memory 的使用範圍有 page 跟 sg list 兩種版本.
  - sync 還分成 for_cpu 跟 for_device 兩種. 

- DMA mapping API

  - map 包裝後有 single, page, sg list 三種.
  - sync 包裝後有 single, single_range, sg list 三種.
  - alloc, free 有 coherent 跟 noncoherent 版本, mmap 只有 coherent 版本.

* all source code at include/linux/dma-mapping.h

DMA mapping ops::

    struct dma_map_ops
    
    - alloc, free
    - mmap
    - get_sgtable
    - {map, unmap}_{page, sg}
    - sync_{single, sg}_for_{cpu, device}
    
DMA mapping API::

    - dma_{map, unmap}_single() => ops->{map, unmap}_page()
    - dma_{map, unmap}_page()   => ops->{map, unmap}_page()
    - dma_{map, unmap}_sg()     => ops->{map, unmap}_sg()

    - dma_sync_single_[range]_for_{cpu, device}() => ops->sync_single_for_{cpu, device}()
    - dma_sync_sg_for_{cpu, device}()             => ops->sync_sg_for_{cpu, device}()
    
    - dma_alloc_coherent()    => ops->alloc()
    - dma_free_coherent()     => ops->free()
    - dma_alloc_noncoherent() => not implemented or same as coherent version.
    - dma_free_noncoherent()  => not implemented or same as coherent version.
    
    - dma_mmap_coherent()     => ops->mmap()
    - dma_mmap_writecombine() => ops->mmap()
    - dma_get_sgtable()       => ops->get_sgtable()

DMA map internal
----------------

目前猜測:

    在沒有 IOMMU 環境下, ``dma_map_*()`` 的行為, 只是去做 cache line 的操作, 或者在加上對 memory type 的設定.
    但應該不會有 memory allocation 以及在 page table 建立新的 memory mapping 的行為發生.

這邊 trace 的是 ARM 裏面四套 DMA mapping ops 實作的其中一套, ``arm_dma_ops``.
`arm_dma_ops`` 的 ``map_page()`` 函式裏面使用了 ARM 的 outercache API 去做 cache invalidate 或 clean 的操作.

trace::

    dma_map_single() => dma_map_single_attrs() 
    => ops->map_page()
       arm_dma_ops.map_page = arm_dma_map_page()
    => __dma_page_cpu_to_dev() 

    => outer_clean_range() // outercache API for ARM
    => outer_cache.clean_range();

``arm_dma_map_page()`` calls ``outer_clean_range()`` or ``outer_inv_range()``, depend on direction of DMA.

outercache
~~~~~~~~~~

source::

    arch/arm/include/asm/outercache.h

API

    outercache 支援 invalidate, clean, flush, sync 四種類型運算
    前三種運算支援部份 range() 的操作, invalidate 跟 flush 也支援 all() 的操作.

implementations::

    // 1. L210/L220 cache controller support
    // arch/arm/mm/cache-l2x0.c
    l2x0_clean_range() => write to HW register L2X0_CLEAN_LINE_PA, for each cache line.
    l2x0_inv_range()   => write to HW register L2X0_INV_LINE_PA, for each cache line.
    l2x0_flush_range() => clean + invalidate

    // 2. Tauros2 L2 cache controller support (Marvell Semiconductor)
    // arch/arm/mm/cache-tauros2.c
    tauros2_clean_range() => call tauros2_clean_pa(), for each cache line.
    tauros2_inv_range()   => call tauros2_inv_pa(), for each cache line.
    tauros2_flush_range() => call tauros2_clean_inv_pa(), for each cache line.

    // before ARMv7?
    tauros2_clean_pa()    : __asm__("mcr p15, 1, %0, c7, c11, 3" : : "r" (addr));
    tauros2_inv_pa()      : __asm__("mcr p15, 1, %0, c7, c7, 3" : : "r" (addr));
    tauros2_clean_inv_pa(): __asm__("mcr p15, 1, %0, c7, c15, 3" : : "r" (addr)); 

DMA alloc internal
------------------

這邊 trace 的是 ``arm_dma_ops`` 的 ``alloc()`` 函式實作.

trace::

    // arch/arm/mm/dma-mapping.c

    - [arm_dma_ops] arm_dma_alloc() => __dma_alloc()
    
      - get_coherent_dma_mask()
      - buf = kzalloc(sizeof(*buf), gfp & ~(__GFP_DMA | __GFP_DMA32 | __GFP_HIGHMEM));
      - buf->allocator = {cma_allocator, simple_allocator, remap_allocator, pool_allocator}
      - addr = buf->allocator->alloc(&args, &page); // 所有 __dma_alloc 的參數都傳入 alloc 了 (by args)
    
ARM 裏面有四套 ``arm_dma_allocator``, 也就是上面 trace 到的 ``cma_allocator, simple_allocator, remap_allocator, pool_allocator`` 這四套.

::

    trace struct arm_dma_allocator simple_allocator
    
    // .alloc() operation
    simple_allocator_alloc() 
    => __alloc_simple_buffer() 
    => __dma_alloc_buffer() 
    => alloc_pages()
    
這邊 trace 的是 ``iommu_ops`` 的 ``alloc()`` 函式實作.

::

    // arch/arm/mm/dma-mapping.c
    [iommu_ops] arm_iommu_alloc_attrs() => __iommu_alloc_atomic()
    => addr = __alloc_from_pool(size, &page);
       => gen_pool_alloc(atomic_pool, size);
    => *handle = __iommu_create_mapping(dev, &page, size);
    
Misc
----

programming technique for op abstraction::

    // 1. dma op 的軟體包裝方式
    
    // include/linux/dma-mapping.h
    struct dma_map_ops {
        void* (*alloc)(struct device *dev, size_t size,
    }
    
    static inline struct dma_map_ops *get_dma_ops(struct device *dev);
    
    static inline void *dma_alloc_attrs(struct device *dev, size_t size,
                           dma_addr_t *dma_handle, gfp_t flag, struct dma_attrs *attrs){
        struct dma_map_ops *ops = get_dma_ops(dev);
        cpu_addr = ops->alloc(dev, size, dma_handle, flag, attrs);
        return cpu_addr;
    }
    
    // 2. arm 實作的 get_dma_ops
    
    // arch/arm/include/asm/dma-mapping.h
    static inline struct dma_map_ops *get_dma_ops(struct device *dev){
        if (dev && dev->archdata.dma_ops)
            return dev->archdata.dma_ops;
        return &arm_dma_ops;
    }
    
source file::

    arch/arm/common/dmabounce.c: dma bounce, dma memory 有限制時的一種特殊技巧的實作.
    include/linux/pci-dma-compat.h

    - 讓有實作 dma op 的 pci device, 使用 pci compatible 介面處理 dma
    - pci_alloc_consistent(pci_device, ...) => dma_alloc_coherent(pci_device->dev, ...)

source code::

    // 讓有實作 dma op 的 pci device, 使用 pci compatible 介面處理 dma
    
    // include/linux/pci-dma-compat.h
    static inline void * pci_alloc_consistent(struct pci_dev *hwdev, size_t size,
                 dma_addr_t *dma_handle)
    {
        return dma_alloc_coherent(hwdev == NULL ? NULL : &hwdev->dev, size, dma_handle, GFP_ATOMIC);
    }
    
other reference
---------------

- CMA allocator: https://events.linuxfoundation.org/images/stories/pdf/lceu2012_nazarwicz.pdf
- gen_pool: general purpose allocator (Uses for this includes on-device uncached memory)
    
  - https://lwn.net/Articles/125842/
