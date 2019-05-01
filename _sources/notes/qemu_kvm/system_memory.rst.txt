QEMU global var ``system_memory``
=================================

source code::
    
    ./exec.c
    ./memory.c

global variable::

    1. static MemoryRegion *system_memory;
    2. AddressSpace address_space_memory;
       extern AddressSpace address_space_memory; // in include/exec/address-spaces.h
    3. static QTAILQ_HEAD(, AddressSpace) address_spaces = QTAILQ_HEAD_INITIALIZER(address_spaces);

set value::
    
    // 1. system_memory && 2. address_space_memory
    static void memory_map_init(void) {
        system_memory = g_malloc(sizeof(*system_memory));
        memory_region_init(system_memory, NULL, "system", UINT64_MAX);
        address_space_init(&address_space_memory, system_memory, "memory");
    }

    // 3. address_spaces
    // 當 AddressSpace 使用 address_space_init() 初始化自己時, 就會順便把自己放入 address_spaces 這個 Tail Queue 當中
    void address_space_init(AddressSpace *as, MemoryRegion *root, const char *name){
        // init code of as
            // as->(root, name) = (root, name)
            // as->current_map; g_new() + flatview_init()
            // ...

        // insert as to Tail Queue address_spaces 
        QTAILQ_INSERT_TAIL(&address_spaces, as, address_spaces_link);

        memory_region_update_pending |= root->enabled;
        memory_region_transaction_commit();
    }
    AddressSpace *address_space_init_shareable(MemoryRegion *root, const char *name);

    void address_space_destroy(AddressSpace *as):

get value::

    // 1. system_memory
    MemoryRegion *get_system_memory(void){
        return system_memory;
    }
    void cpu_exec_init(CPUState *cpu, Error **errp){
        cpu->memory = system_memory;
    }

    // 2. address_space_memory
    void cpu_physical_memory_rw(hwaddr addr, uint8_t *buf, int len, int is_write){
        address_space_rw(&address_space_memory, addr, MEMTXATTRS_UNSPECIFIED, buf, len, is_write);
    }
    void cpu_flush_icache_range(hwaddr start, int len);
        // calls cpu_physical_memory_write_rom_internal()
    cpu_physical_memory_map()/cpu_physical_memory_unmap(): address_space_map/unmap(&address_space_memory)

    bool cpu_physical_memory_is_io(hwaddr phys_addr){
        MemoryRegion mr = address_space_translate(&address_space_memory, phys_addr, &phys_addr, &l, false);
        return !(memory_region_is_ram(mr) || memory_region_is_romd(mr));
    }
    // 看起來操作 address_space_memory 的函式, 基本上都是操作 struct AddressSpace 的函式的再包裝, 而且大部份都有使用
    // AddressSpace to MemoryRegion 的 API: address_space_translate()

    // 3. address_spaces
    // QTAILQ_FOREACH + address_spaces
    static AddressSpace *memory_region_to_address_space(MemoryRegion *mr)
    void memory_region_transaction_commit(void);
    void memory_region_sync_dirty_bitmap(MemoryRegion *mr);
    static void memory_region_update_coalesced_range(MemoryRegion *mr);

    void memory_listener_register(MemoryListener *listener, AddressSpace *filter);

    void mtree_info(fprintf_function mon_printf, void *f);

    // 4. struct AddressSpace API

Misc
----

- address_space_memory: 是系統 memory 用的 AddressSpace

  - address_space_rw() parameter: 當 MMIO 發生時, 會從 address_space_memory 搜尋對應的 MemoryRegion

- system_memory: 是系統 memory 的 root MemoryRegion

  - memory_region_add_subregion_*() parameter: device 產生 MMIO 的 MemoryRegion, 會放入 system_memory 當中

- address_spaces: 是所有 AddressSpace 的集合, 包含 address_space_memory

  - address_space_init(): 當 AddressSpace init 時, 就會被放入 address_spaces.
  - memory_region_to_address_space(): 尋找 MemoryRegion 隸屬的 AddressSpace, 會從 address_spaces 做搜尋.

- memory_listeners
