QEMU system init
================

Basic
-----

- entry point: ``vl.c::main()``

  - machine initialization

    - CLI option ``-M <machine>`` decides it.
    - init function: ``machine_class->init(machine_class)``
    - initialize motherboard or development board.
    - initialize cpu, memory, bus, devices for motherboard

  - misc device init: ``device_init_func()``

    - CLI option ``-device <device>,<options>``.
    - for each ``-device`` in CLI options, call ``device_init_func()`` to initialize it.
    - ``device_init_func()`` calls ``dev->realize()`` to init device.
    - e.g. ivshmem(Inter-VM Share Memory) is initialized by ``ivshmem_realize()``.

  - QEMU IO thread runs into main loop: ``main_loop()``

- multi-threading

  - CPU initialization will create (``pthread_create()``) multiple threads, which is called QEMU vCPU thread.
    The thread number is equals to vCPU number.
  - More: :doc:`qemu_thread_event` , :doc:`qemu_misc_note`

Machine Init
------------

all machines are initialized by class ``init()`` member function.

``main()`` runs it::
  
    // vl.c::main()
    current_machine = MACHINE(object_new(object_class_get_name(OBJECT_CLASS(machine_class))));
    machine_class->init(current_machine)

options
~~~~~~~

``machine_class->init()`` depends on CLI option: ``-M <machine>``, and default machine of ``qemu-system-x86_64`` is i440FX.

``machine_class->init()`` common options
  
- x86 i440FX: ``pc_init1()`` at ``hw/i386/pc_piix.c``
- x86 q35: ``pc_q35_init()`` at ``hw/i386/pc_q35.c``
- arm virt: ``machvirt_init()`` at ``hw/arm/virt.c``
- arm vexpress-a15: ``vexpress_common_init()`` => ``a15_daughterboard_init()`` at ``hw/arm/vexpress.c``
- find more options by QEMU help::
    
    $ qemu-system-arm -M help

    virt            QEMU 2.8 ARM Virtual Machine (alias of virt-2.8)
    vexpress-a15    ARM Versatile Express for Cortex-A15
    ...
    
    # x86 -M help shows pc and q35

internal
~~~~~~~~

Using i440FX as example. i440FX's init() is ``pc_init1()``.

``pc_init1()`` initialize init cpu, memory, bus, devices for motherboard i440FX.

- CPU: ``pc_cpus_init()``. it use ``pthread_create()`` to create vCPU threads in KVM Mode, and vCPU threads use ``ioctl(KVM_CREATE_VCPU)`` to runs guest OS.
- Memory: ``pc_memory_init()``. it initialize global variable ``system_memory``. 
- VGA: ``pc_vga_init()``, it calls realize function of VGA device (e.g. ``pci_cirrus_vga_realize()``)
- NIC: ``pc_nic_init()``, it calls realize function of NIC device (e.g. ``pci_e1000_realize()``)
- IDE: ``pci_piix3_ide_init()``, init disk ...

Device Init
-----------

all QEMU devices is initialized by ``realize()`` member function, which is triggered by setter of ``realized`` property.

- example source code (i440FX init function initialize NIC)::

    // pc_init1() => pc_nic_init() => pci_e1000_realize()

    pc_nic_init() => pci_nic_init_nofail() => qdev_init_nofail()
    => object_property_set_bool(OBJECT(dev), true, "realized", &err); // set ``realized`` property to true.
    => pci_e1000_realize(); // e1000 NIC's realize function.

options
~~~~~~~

NIC common options

- e1000: ``pci_e1000_realize()`` at ``hw/net/e1000.c``
- ne2k: ``pci_ne2000_realize()`` at ``hw/net/ne2000.c``
- rtl8139: ``pci_rtl8139_realize()`` at ``hw/net/rtl8139.c``
- virtio: ``virtio_net_device_realize()`` at ``hw/net/virtio-net.c``
- find more options

  - `QEMU Wiki <https://en.wikibooks.org/wiki/QEMU/Devices/Network>`_
  - QEMU help::

      $ qemu-system-x86_64 -device help

      Network devices:
      name "e1000", bus PCI, alias "e1000-82540em", desc "Intel Gigabit Ethernet"
      name "e1000-82544gc", bus PCI, desc "Intel Gigabit Ethernet"
      name "ne2k_isa", bus ISA
      name "ne2k_pci", bus PCI
      name "pcnet", bus PCI
      name "rtl8139", bus PCI
      ...

  - searching source code::

      $ grep -Rn hw/net/ -e "realize"

      hw/net/e1000.c:1587:static void pci_e1000_realize(PCIDevice *pci_dev, Error **errp)  
      hw/net/ne2000.c:720:static void pci_ne2000_realize(PCIDevice *pci_dev, Error **errp)
      hw/net/ne2000-isa.c:62:static void isa_ne2000_realizefn(DeviceState *dev, Error **errp)
      hw/net/pcnet-pci.c:281:static void pci_pcnet_realize(PCIDevice *pci_dev, Error **errp)
      ...

Code detail and Note
--------------------

1. QEMU implements OOP in C, QEMU name it's OO model QOM(QEMU Object Model). 

   - each classes has a ``TypeInfo`` struct, which define some metadata like class name and parent classes.
   - more about QOM: :doc:`./QOM1`

2. QEMU devices are implemented by QDev. QDev is reimplemented on the QOM currently.

   - QDev is a class, and it is a base class of all QEMU devices.
   - motherboard, cpu, vga, nic inherit QDev.
   - all QDev device is initialized by ``realize()`` member function.
     It is triggered by setter of ``realized`` property. ``object_property_set_bool(OBJECT(dev), true, "realized", &err);``

     - detail: http://nairobi-embedded.org/035_qom_and_device_creation.html#object-constructor-callback-mechanism

3. why ``machine_class->init()`` calls ``pc_init1()``::

    // in QEMU v2.7

    // expand MACRO DEFINE_I440FX_MACHINE, which define motherboard class "pc-i440fx-2.7" (QEMU class by QEMU Object Model)

    DEFINE_I440FX_MACHINE(v2_7, "pc-i440fx-2.7", NULL, pc_i440fx_2_7_machine_options);

    // DEFINE_PC_MACHINE(v2_7, "pc-i440fx-2.7", pc_init_v2_7, pc_i440fx_2_7_machine_options);
    
    type_init(pc_machine_init_v2_7)
    static void pc_machine_init_v2_7(void)
    {
        type_register(&pc_machine_type_v2_7);
    }
    static const TypeInfo pc_machine_type_v2_7 = {
        .name = "pc-i440fx-2.7",
        .parent = "generic-pc-machine",
        .class_init = pc_machine_v2_7_class_init,
    }
    pc_machine_v2_7_class_init(ObjectClass *oc, void *data){
        MachineClass *mc = MACHINE_CLASS(oc);
        pc_i440fx_2_7_machine_options(mc);
        mc->name = "pc-i440fx-2.7"
        mc->init = pc_init_v2_7;  // mc->init() is pc_init_v2_7(), which calls pc_init1()
    }
    pc_init_v2_7(MachineState *machine){
      ... // call compat

      pc_init1()
    }

4. QEMU has GLib dependecies. it uses GLib data type, data structures, and event loop.

   - :doc:`./qemu_glib`

reference
---------

- `QEMU wiki - Features/Q35 <http://wiki.qemu.org/Features/Q35>`_
- `QEMU wiki - Documentation/Platforms/PC <http://wiki.qemu.org/Documentation/Platforms/PC>`_,
  `QEMU wiki - Documentation/Platforms/ARM <http://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm>`_
- `wiki - Intel 440FX <https://en.wikipedia.org/wiki/Intel_440FX>`_,
  `wiki - PIIX <https://en.wikipedia.org/wiki/PCI_IDE_ISA_Xcelerator#PIIX3>`_
- ``qemu-system-x86_64 -M help``, ``qemu-system-arm -M help``
