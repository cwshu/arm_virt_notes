Hardware IO in linux driver
===========================

.. contents:: Table of Contents

Outline
-------

1. Functions that handle resource allocation for memory regions::

     struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);
     void release_mem_region(unsigned long start, unsigned long len);
     int check_mem_region(unsigned long start, unsigned long len);

     # allocated memory is shown at /proc/iomem

2. ioremap remaps a physical address range into the processor’s virtual address
   space, making it available to the kernel. iounmap frees the mapping when it is no
   longer needed. ::

     #include <asm/io.h>
     void *ioremap(unsigned long phys_addr, unsigned long size);
     void *ioremap_nocache(unsigned long phys_addr, unsigned long size);
     void iounmap(void *virt_addr);

3. Accessor functions that are used to work with I/O memory.::

     #include <asm/io.h>
     unsigned int ioread8(void *addr);
     unsigned int ioread16(void *addr);
     unsigned int ioread32(void *addr);
     void iowrite8(u8 value, void *addr);
     void iowrite16(u16 value, void *addr);
     void iowrite32(u32 value, void *addr);

4. "Repeating" versions of the I/O memory primitives.::

     void ioread8_rep(void *addr, void *buf, unsigned long count);
     void ioread16_rep(void *addr, void *buf, unsigned long count);
     void ioread32_rep(void *addr, void *buf, unsigned long count);
     void iowrite8_rep(void *addr, const void *buf, unsigned long count);
     void iowrite16_rep(void *addr, const void *buf, unsigned long count);
     void iowrite32_rep(void *addr, const void *buf, unsigned long count);

request_mem_region
------------------

a. resource

  iomem_resource, ioport_resource::
  
      // kernel/resource.c
      struct resource iomem_resource = {
          .name   = "PCI mem",
          .start  = 0, 
          .end    = -1,
          .flags  = IORESOURCE_MEM,
      };
      EXPORT_SYMBOL(iomem_resource);   

  struct resource::

    // include/linux/ioport.h
    struct resource {
        // memory address?
        resource_size_t start;
        resource_size_t end;

        const char *name;
        unsigned long flags;
        unsigned long desc;
        
        // left-child right-sibling tree
        struct resource *parent, *sibling, *child;
    };

b. request API

   - request_mem_region() wrapper::

       #define request_mem_region(start,n,name) __request_region(&iomem_resource, (start), (n), (name), 0) 
       #define request_region(start,n,name)     __request_region(&ioport_resource, (start), (n), (name), 0)

   - __request_region()
   
     - alloc_resource(): allocate struct resource and return
     - __request_resource(): add new resource to the child of parent resource.

   - alloc/free resource
   
     - alloc_resource(): allocate struct resource and return
     
       - (1) allocate from freelist(bootmem_resource_free) + memset()
       - (2) kzalloc()
   
     - free_resource(): (1) kfree() (2) free to freelist(bootmem_resource_free)
     
       - PageSlab(virt_to_head_page(res))

c. ``/proc/iomem``::

     // kernel/resource.c
     __initcall(ioresources_init);
     static int __init ioresources_init(void)
     {
         proc_create("ioports", 0, NULL, &proc_ioports_operations);
         proc_create("iomem", 0, NULL, &proc_iomem_operations);
         return 0;
     }
         
     static const struct file_operations proc_iomem_operations = {
         .open       = iomem_open,
         .read       = seq_read,
         .llseek     = seq_lseek,
         .release    = seq_release,
     };

     static const struct seq_operations resource_op = {
         .start  = r_start,
         .next   = r_next,
         .stop   = r_stop,
         .show   = r_show,
     };

d. reserve at init API

   ::

     // kernel/resource.c
     __setup("reserve=", reserve_setup);
     #define MAXRESERVE 4
     static int __init reserve_setup(char *str)
     {
         static int reserved;
         static struct resource reserve[MAXRESERVE];

         for (;;) {
             unsigned int io_start, io_num;
             int x = reserved;

             if (get_option (&str, &io_start) != 2)
                 break;
             if (get_option (&str, &io_num)   == 0)
                 break;
             if (x < MAXRESERVE) {
                 struct resource *res = reserve + x;
                 res->name = "reserved";
                 res->start = io_start;
                 res->end = io_start + io_num - 1;
                 res->flags = IORESOURCE_BUSY;
                 res->desc = IORES_DESC_NONE;
                 res->child = NULL;
                 if (request_resource(res->start >= 0x10000 ? &iomem_resource : &ioport_resource, res) == 0)
                     reserved = x+1;                                                                                                    
              }
         }
         return 1;
     }


resource::

    kernel/resource.c
    - struct resource ioport_resource, iomem_resource
    - proc_ioport, proc_iomem // seq_operations traverse resource tree?
    - request, release, find
    - devm version
    - __init reserve_setup


    include/linux/resource.h
    include/uapi/linux/resource.h

bootmem::

    mm/bootmem.c
    include/linux/bootmem.h

    // More
    drivers/base/memory.c: The probe routines leave the pages reserved, just as the bootmem code does.
    drivers/base/dma-contiguous.c: called by arch specific code once the early allocator (memblock or bootmem)
    drivers/base/platform.c: #include<linux/bootmem.h>

ioremap internal
----------------

callstack (x86)::

    void __iomem *ioremap() at arch/x86/include/asm/io.h
    void __iomem *ioremap_nocache() at arch/x86/mm/ioremap.c
    void __iomem *__ioremap_caller() at arch/x86/mm/ioremap.c
    int ioremap_page_range() at lib/ioremap.c

callstack (arm64)::

    //

ioremap_page_range(addr, end, phys_addr, prot)::

    // lib/ioremap.c
    map vaddr=(addr, end) to paddr=(phys_addr, phys_addr + end-addr), pgtable permission is prot.
    use vaddr to find correct pte(range of ptes), than set pte = (paddr, prot).

    compare to remap_pfn_range() in mm/memory.c

misc::

    pgd = pgd_offset_k(addr)              
    pud = pud_alloc(&init_mm, pgd, addr);
    pmd = pmd_alloc(&init_mm, pud, addr);
    pte = pte_alloc_kernel(pmd, addr)     => pte_offset_kernel(pmd, addr)

    // include/asm-generic/pgtable.h
    next = pgd_addr_end(addr, end)
    next = pud_addr_end(addr, end);
    next = pmd_addr_end(addr, end);

    pud_set_huge(pud, phys_addr + addr, prot)
    pmd_set_huge(pmd, phys_addr + addr, prot)


----

in Linux 4.7::

    include/asm-generic/io.h
    include/asm-generic/iomap.h

    include/linux/ioport.h
    include/linux/io-mapping.h ??

    include/linux/device.h
    drivers/base/devres.c
    lib/devres.c

devm: device resource management
--------------------------------

devm_ioremap_resource() consists of

- devm_request_mem_region() => __request_region() // p.s. request_mem_region() => __request_region()
- devm_ioremap() => ioremap()

More about devm_request_mem_region()

- => __devm_request_region() => devres_alloc() + __request_region()

Reference
---------

device resource management

- device resource management: https://lwn.net/Articles/215996/
- The managed resource API: https://lwn.net/Articles/222860/
- https://www.kernel.org/doc/Documentation/driver-model/devres.txt

reference

- LDD3 ch9.4: http://www.makelinux.net/ldd3/chp-9-sect-4
