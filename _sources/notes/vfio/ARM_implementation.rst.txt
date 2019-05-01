VFIO ARM resource
=================

- `[2014] VOS - Platform Device Assignment to KVM-on-ARM Virtual Machines via VFIO <http://www.fp7-save.eu/papers/EUC2014B.pdf>`_
- `[2014] KVM Platform Device Passthrough <http://www.linux-kvm.org/images/a/a8/01x04-ARMdevice.pdf>`_
- `[2013] (freescale) VFIO for Platform Devices <https://www.linux-kvm.org/images/0/09/Kvm-forum-2013-vfio_platform.pdf>`_
- `Linaro 找到的 add new device to VFIO PLATFORM 教學 <https://wiki.linaro.org/LEG/Engineering/Virtualization/VFIOPlatformHowToIntegrateNewDevice>`_

Terminology
-----------

- ``VFIO_PLATFORM``: VFIO support of device under the platform bus.

    Platform bus means platform dependent bus. 
    In Linux, platform devices are registered at ``/sys/bus/platform``.

    Compared to it, there are some common platform independent buses like pci, usb, and amba. 
    They are also at ``/sys/bus/`` directory. 
    
    see ``sysfs`` for more information.

VFIO_PLATFORM for DMA Controller (PL330)
----------------------------------------

- `Testing VFIO_PLATFORM with the PL330 DMA Controller <http://www.virtualopensystems.com/en/solutions/guides/vfio-on-arm/>`_

  - VOS 寫了 ARM platform 的 VFIO support, 支援 PL330 DMA Controller 跟 IOMMU.
  - 測試環境是 emulator (ARM Foundation Model)
  - patch - `VFIO support for platform devices <https://lwn.net/Articles/584940/>`_

    - find ``VFIO_PLATFORM`` code in mainline linux (linux 4.7 is ok): ``grep -R drivers/vfio -e 'Antonios Motakis'``

  - `VFIO_PLATFORM userspace test driver <https://github.com/virtualopensystems/vfio-host-test>`_
  
- `VFIO_PLATFORM paper <http://www.fp7-save.eu/papers/EUC2014B.pdf>`_

VFIO_PLATFORM for Calxeda xgmac ethernet driver
-----------------------------------------------

- `[2014] KVM Platform Device Passthrough <http://www.linux-kvm.org/images/a/a8/01x04-ARMdevice.pdf>`_

  - Calxeda xgmac ethernet driver
  - KVM VFIO irqfd

- xgmac device pass through source code 疑似有進 QEMU 跟 Linux kernel mainline.

  - 原 driver source code: ``drivers/net/ethernet/calxeda/xgmac.c``
  - QEMU patch?:
   
    - (1) [PATCH v15 00/10] KVM platform device passthrough: https://www.spinics.net/lists/kvm-arm/thrd15.html#14541
    - (2) https://patches.linaro.org/patch/43179/

  - [Question] Where is linux kernel patch? is it exist?
  - vfio platform xgmac reset module (KVM) https://patchwork.kernel.org/patch/6358861/ 
    
    - ``drivers/vfio/platform/reset/vfio_platform_calxedaxgmac.c``

Some kernel patches
-------------------
- `Merge tag 'vfio-v4.2-rc1' of git://github.com/awilliam/linux-vfio <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b779157dd3db6199b50e7ad64678a1ceedbeebcf>`_

  - add reset hooks and Calxeda xgmac reset for vfio-platform (Eric Auger)
  - enable vfio-platform for ARM64 (Eric Auger)
  - tag Baptiste Reynal as vfio-platform sub-maintainer (Alex Williamson)
  - ... 

- [linux-v4.2] `vfio: platform: add the VFIO PLATFORM module to Kconfig <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=53161532394b3b3c7e1ec9c80658edd75446ac77>`_
- [linux-v4.1] `vfio: amba: VFIO support for AMBA devices <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=36fe431f2811fa3b5fed15d272c585d5a47977aa>`_
- `Merge tag 'vfio-v3.19-rc1' of git://github.com/awilliam/linux-vfio <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cc669743a39e3f61c9ca5e786e959bf478ccd197>`_

  - Enable iommu-type1 for ARM SMMU (Will Deacon)

KVM PCIe/MSI Passthrough on ARM/ARM64
-------------------------------------

`Eric Auger, KVM PCIe/MSI Passthrough on ARM/ARM64 <https://www.linaro.org/blog/core-dump/kvm-pciemsi-passthrough-armarm64/>`_

- ARM-based system require special support for MSI in the context of VFIO passthrough (ARM server, PCI MSI)
- chapters

  - ARM MSI Controllers: GICv2m and GICv3 ITS
  - KVM PCI/MSI passthrough, x86/ARM Differences
    
    - On x86, MSI write transactions hit special 1MB phyaddr. It's APIC config space and not DRAM, so it bypass IOMMU. It doesn't need IOMMU mapping.
    - On ARM, however, MSI write transactions is normal MMIO region. Therefore an IOMMU mapping must exist.
    - The goal: create the needed IOMMU mappings for MSI write transactions. eventually reach hardware MSI frame.
  
  - Assigned device MSI Setup: VFIO Legacy Implementation for x86, and Requested adaptation for ARM
  - Interrupt Safety: guest cannot trigger MSIs that correspond to interrupt IDs of devices belonging to host or other guest.
  - Conclusions

    - GICv2m MSI controllers will require users to load VFIO with the ``allow_unsafe_interrupts`` parameter.
    - GICv3 ITS platforms will work with VFIO without any additional parameters.

- patches: 
  `Linux kernel <https://lkml.org/lkml/2016/2/12/47>`_,
  `QEMU <http://lists.gnu.org/archive/html/qemu-arm/2016-01/msg00444.html>`_

Other Resource
--------------

- Direct Device Assignment for Untrusted Fully-Virtualized Virtual Machines, IBM Research, 2008
- Introduction on performance analysis and profiling methodologies for KVM on ARM virtualization
