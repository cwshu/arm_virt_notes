Ubuntu Server on qemu-system-aarch64/kvm
========================================

Preparation
-----------
1. Linaro QEMU_EFI

   - `download <https://releases.linaro.org/components/kernel/uefi-linaro/15.12/release/qemu64/QEMU_EFI.fd>`_
   - `latest release <https://releases.linaro.org/components/kernel/uefi-linaro/latest/>`_
   - `introduction <https://wiki.linaro.org/LEG/UEFIforQEMU>`_

     - support QEMU/arm machvirt machine type (``-M virt``) both in AArch64 and AArch32.
     - support virtio block, network and SCSI devices.
     - need QEMU >= 2.2 and built for aarch64-softmmu/arm-softmmu
     - QEMU_EFI.fd is Tianocore EDK2's ArmVirtPkg/ArmVirtQemu.dsc (https://github.com/tianocore/edk2)
       
2. Ubuntu Server ISO for AArch64 

   - `16.04.2 (ubuntu xenial) release <http://cdimage.ubuntu.com/releases/16.04.2/release/>`_

Installation
------------
- QEMU running command:: 

    qemu-system-aarch64 \
        -enable-kvm -cpu host \
        -M virt -bios QEMU_EFI.fd \
        -m 1024 -smp 4 \
        -nographic \
        -drive id=cdrom1,media=cdrom,if=none,file=<ubuntu_install_iso> \
        -device virtio-scsi-device -device scsi-cd,drive=cdrom1 \
        -drive id=hd0,media=disk,if=none,format=qcow2,file=<ubuntu_qcow2_image> \
        -device virtio-blk-device,drive=hd0 \
        -netdev user,id=user -device virtio-net-device,netdev=user

    # input:
    #   Ubuntu ISO path
    #   Ubuntu qcow2 path

- Ubuntu installation

.. _boot_uefi_issue:

boot up issue(UEFI)
~~~~~~~~~~~~~~~~~~~

Booting up is blocking at EFI menu for arm64 rootfs of ubuntu, debian, and fedora.

2 solutions

1. execute EFI bootup script in EFI shell ``EFI\ubuntu\grubaa64.efi``
2. write a EFI script ``startup.nsh`` to execute EFI bootup script

`EFISTUB path <https://wiki.debian.org/UEFI#Booting_a_UEFI_machine_normally>`_

Running and Usage
-----------------
- QEMU running command::

    qemu-system-aarch64 \
        -enable-kvm -cpu host \
        -M virt -bios QEMU_EFI.fd \
        -m 1024 -smp 4 \
        -nographic \
        -drive id=hd0,media=disk,if=none,format=qcow2,file=<ubuntu_qcow2_image> \
        -device virtio-blk-device,drive=hd0 \
        -netdev user,id=user -device virtio-net-device,netdev=user

    # input:
    #   Ubuntu qcow2 path

- guest network: dhcp::

    sudo dhclient -v enp0s1
