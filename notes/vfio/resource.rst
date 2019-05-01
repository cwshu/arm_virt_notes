VFIO resource
=============

introduction

- `[2016] An Introduction to PCI Device Assignment with VFIO <http://www.linux-kvm.org/images/5/54/01x04-Alex_Williamson-An_Introduction_to_PCI_Device_Assignment_with_VFIO.pdf>`_
- `[2012] VFIO: A User's Perspective <https://www.linux-kvm.org/images/b/b4/2012-forum-VFIO.pdf>`_
- `(2012) LWN - Safe device assignment with VFIO <https://lwn.net/Articles/474088/>`_

other intro

- `[2016] IBM LTC China - VFIO introduction <https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/20160605>`_
- `[2013] IBM LTC China - VFIO 簡介 <https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/vfio>`_

More

- ``Documentation/vfio.txt``
- `vfio 内核实现分析 (1) ~ (7) <http://blog.csdn.net/WangyueSongshan/article/details/50363714>`_

Kernel Patches
--------------

- (2012, linux-v3.6) 1st VFIO patches

  - `VFIO: bare-metal safe access to devices from userspace drivers <https://kernelnewbies.org/Linux_3.6#head-edf6c6d47bba170502ccfd4706419cd4f2ce0e30>`_
  - 4 commit: VFIO core, IOMMU, PCI device driver, doc

- (應該不太需要了) patch before mainline kernel: `VFIO core framework <http://lwn.net/Articles/473975/>`_, `code <http://www.mail-archive.com/kvm@vger.kernel.org/msg65422.html>`_

VFIO_PLATFORM

- see :doc:`/notes/vfio/ARM_implementation`

VFIO no-IOMMU
~~~~~~~~~~~~~

- `Merge tag 'vfio-v4.4-rc1' of git://github.com/awilliam/linux-vfio <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=934f98d7e8123892bd9ca8ea08728ee0784e6597>`_
  
  - No-IOMMU interface (Alex Williamson) 

- [linux-v4.5] `vfio: Include No-IOMMU mode <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=03a76b60f8ba27974e2d252bc555d2c103420e15>`_
- [linux-v4.8] `vfio: platform: support No-IOMMU mode <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9698cbf0bea6b9f5c3190ce97bdf8963c0148671>`_
- `Merge tag 'vfio-v4.8-rc1' of git://github.com/awilliam/linux-vfio <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e55884d2c6ac3ae50e49a1f6fe38601a91181719>`_

  - Enable no-iommu mode for platform devices (Peng Fan)
  - vfio: platform: support No-IOMMU mode

Misc kernel patches
~~~~~~~~~~~~~~~~~~~
- [linux-v4.4] Support for VT-d posted interrupts. Used by KVM and VFIO

  - `virt: IRQ bypass manager <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f73f8173126ba68eb1c42bd9a234a51d78576ca6>`_

- [linux-v3.13] `kvm: Add VFIO device <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec53500fae421e07c5d035918ca454a429732ef4>`_

  - KVM VFIO interaction by KVM-VFIO device

- [linux-v3.12] `vfio-pci: PCI hot reset interface <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8b27ee60bfd6bbb84d2df28fa706c5c5081066ca>`_
- [linux-v3.6] `VFIO: bare-metal safe access to devices from userspace drivers <https://kernelnewbies.org/Linux_3.6#head-edf6c6d47bba170502ccfd4706419cd4f2ce0e30>`_
