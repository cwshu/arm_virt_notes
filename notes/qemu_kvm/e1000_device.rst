QEMU e1000 Device Emulation
===========================

e1000 keynote
-------------

Interface between QEMU e1000 device and other modules.

- e1000 device to network backend (e.g. slirp or tap)

  - e1000 send packets to network backend by ``qemu_send_packet()``.
    Slirp backend send packets by ``send()`` syscall.
  - e1000 recieve packets from network backend by ``NetClientInfo.receive()`` callback.
    Slirp backend recieve packets by ``poll()`` or ``recv()`` syscall.

- e1000 device to guest OS driver

  - e1000 device handles MMIO access by ``MemoryRegion.ops.{read(), write()}``
  - e1000 device send virtual interrupt to guest OS by ``set_ics()`` or ``set_interrupt_cause()``
    ( based on ``pci_set_irq()``. All PCI devices can use this API to send VIRQ. )
  - e1000 HW registers
    
    - e1000 device access them by access ``mac_reg`` register array.
    - Guest driver access them by MMIO, e1000 emulates register access: ``MemoryRegion.ops.write() => macreg_writeops[reg]()``

  - e1000 DMA descriptors (TX/RX ring)

    - Pointed from e1000 HW registers: ``mac_reg[TDH]``, ``mac_reg[RDH]`` ...
    - Guest driver use simple memory access, device use emulated DMA accessing.

e1000 相關 reference

- `e1000 datasheet <https://pdos.csail.mit.edu/6.828/2014/readings/hardware/8254x_GBe_SDM.pdf>`_

e1000 TX
--------

Whenever guest driver accesses ``TCTL`` and ``TDT`` HW registers, it will trigger e1000 device to send packet.
QEMU e1000 device copy packets from TX Ring at guest memory, it copies data by DMA emulation.
Then, QEMU send packet by network backend API. 
After sending packet, QEMU send completion interrupt to notify guest OS by KVM virtual interrupt.

QEMU e1000 device implements ``start_xmit()`` to send packets.

e1000 start_xmit() dataflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- DMA emulation: use ``pci_dma_read()`` to copy data from TX Ring to ``struct e1000_tx* tp``.
- send packet by network backend: calls ``qemu_send_packet()`` to send packet from ``struct e1000_tx* tp``.

::
    
    process_tx_desc(E1000State *s, struct e1000_tx_desc *dp)
       // struct e1000_tx *tp = &s->tx;
       // pci_dma_read(d, addr, tp->data + tp->size, bytes); 

    => xmit_seg(s)
       // struct e1000_tx *tp = &s->tx;
    => e1000_send_packet(s, tp->data, tp->size)
    => qemu_send_packet(qemu_get_queue(s->nic), tp->data, tp->size)

How QEMU emulates DMA?

1. emulated DMA access ( ``pci_dma_read()`` ) is equal to guest address space access. ( AddressSpace API ``address_space_rw()`` )
2. target AS is ``PCIDevice.bus_master_as``
3. If target MR is ``system_memory``, QEMU use ``memcpy()`` to emulate it. 
   Address translation from GPA to HVA is processed at translation from AS to MR, it is SW translation. I don't know ``address_space_translate()`` has cache or not.

::

    pci_dma_read(PCIDevice *dev, dma_addr_t addr, void *buf, dma_addr_t len)
    => dma_memory_rw(&dev->bus_master_as, addr, buf, len, DMA_DIRECTION_TO_DEVICE);
    => address_space_rw(&dev->bus_master_as, addr, MEMTXATTRS_UNSPECIFIED, buf, len, false):

    calls memcpy() because mr is system_memory (memory_region alias)

e1000's ``bus_master_as`` is a alias to ``system_memory`` in the experiment.

::

    monitor) info mtree
    address-space: 
      0000000000000000-ffffffffffffffff (prio 0, i/o): bus master container
        0000000000000000-ffffffffffffffff (prio 0, i/o): alias bus master @system 0000000000000000-ffffffffffffffff

e1000 sending completion interrupt
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``start_xmit()`` send packet by ``process_tx_desc()`` and send completion interrupt by ``set_ics()``

``set_ics()`` send VIRQ by ``pci_set_irq()``::

    set_ics(E1000State *s, int index, uint32_t val) 
    => set_interrupt_cause(s, 0, val | s->mac_reg[ICR])
       => pci_set_irq(d, s->mit_irq_level);

    set_interrupt_cause(E1000State *s, int index, uint32_t val)
    // PCIDevice *d = PCI_DEVICE(s);

    // s->mac_reg[ICR] = s->mac_reg[ICS] = val;
    // pending_ints = (s->mac_reg[IMS] & s->mac_reg[ICR]);
    // s->mit_irq_level = (pending_ints != 0);
    => pci_set_irq(d, s->mit_irq_level);

``PCIDevice::pci_set_irq()``::

    pci_set_irq(PCIDevice *pci_dev, int level)
    => pci_irq_handler(pci_dev, pci_intx(pci_dev), level);
       => pci_set_irq_state(): set pci_dev->irq_state to (level<<irq_num)
       => pci_update_irq_status(): turn on/off 1 bit status in dev->config[PCI_STATUS]'s PCI_STATUS_INTERRUPT bit.
       => pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
          // recursively find PCIBus (pci_dev->bus) and parent device (bus->parent_dev), until bus->set_irq exist
          // irq number mapping recursively: irq_num = bus->map_irq(pci_dev, irq_num);
          => bus->set_irq(bus->irq_opaque, irq_num, bus->irq_count[irq_num] != 0);
       
``PCIBus::set_irq()``::

    // pci_bus_irqs() and pci_register_bus() register set_irqs

    // PCI host bridges's set_irqs
    //   i440fx use PIIX:
    //     i440fx_init() register piix3_set_irq():
    //     pci_bus_irqs(b, piix3_set_irq, pci_slot_get_pirq, piix3, PIIX_NUM_PIRQS);
    //   Q35:
    //     pc_q35_init() register ich9_lpc_set_irq()
    //   ARM virt use GPEX:
    //     gpex_host_realize() register gpex_set_irq()

    gpex_set_irq(void *opaque, int irq_num, int level)
    => qemu_set_irq(s->irq[irq_num], level);
       // qemu_set_irq(qemu_irq irq, int level)
       => irq->handler(irq->opaque, irq->n, level);

``qemu_irq->handler``::

    qemu_irq == IRQState, IRQState->handler is qemu_irq_handler

    // for GPEX_HOST in machvirt machine
    // GPEX_HOST's qemu_irq = KVM_ARM_GIC's qemu_irq = kvm_arm_gicv2_set_irq

    machvirt_init()
    => create_gic(VirtMachineState *vms, qemu_irq *pic)
       // create KVM_ARM_GIC, set kvm_arm_gicv2_set_irq to pic
       => kvm_arm_gic_realize()   
          => gic_init_irqs_and_mmio(s, kvm_arm_gicv2_set_irq, NULL);
             // DeviceState->gpios = kvm_arm_gicv2_set_irq
       => pic[i] = qdev_get_gpio_in(gicdev, i);
          // pic = DeviceState->gpios
    
    => create_pcie(VirtMachineState *vms, qemu_irq *pic)
       // create GPEX_HOST, set pic to GPEX_HOST's qemu_irq
       => sysbus_connect_irq(SYS_BUS_DEVICE(dev), i, pic[irq + i]);

Misc
----

``qemu_irq`` misc::

    // kvm_i8259_init(ISABus *bus)
    //   qemu_allocate_irqs(kvm_pic_set_irq, NULL, ISA_NUM_IRQS);
    // /hw/i386/pc_piix.c::pc_init1()
    //   pcms->gsi = 
    //     [kvm_ioapic_in_kernel] qemu_allocate_irqs(kvm_pc_gsi_handler)
    //     qemu_allocate_irqs(gsi_handler)
    //   smi_irq = qemu_allocate_irq(pc_acpi_smi_interrupt)
    // /hw/i386/pc.c::pc_allocate_cpu_irq()
    //   qemu_allocate_irq(pic_irq_request, NULL, 0);
    // i8254.c::kvm_pit_realizefn()
    //   qdev_init_gpio_in(dev, kvm_pit_irq_control, 1);
    // ioapic.c::kvm_ioapic_realize()
    //   qdev_init_gpio_in(dev, kvm_ioapic_set_irq, IOAPIC_NUM_PINS);

