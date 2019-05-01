ArchLinux ARM on odroid c2
==========================

Installation
------------
- installation guide: https://archlinuxarm.org/platforms/armv8/amlogic/odroid-c2
- default login: dhcp over ethernet + ssh, usr/pwd: alarm/alarm

some issue
~~~~~~~~~~
- ext4 metadata checksum

  - Why we use options to format ext4 fs: ``mkfs.ext4 -O ^metadata_csum,^64bit /dev/sdX1``
  - `Re: ODROID-C2 won't boot to Arch but it will to Ubuntu [SOLVED] <https://archlinuxarm.org/forum/viewtopic.php?f=65&t=10560#p52600>`_
  - https://ext4.wiki.kernel.org/index.php/Ext4_Metadata_Checksums

- how to serial console on odroid c2

  - `UART to USB Module Kit <http://www.hardkernel.com/main/products/prdt_info.php?g_code=G134111883934>`_
  - `Serial connection for Odroid <https://ridderbusch.name/post/2016-06-10-serial-for-odroid/>`_

Basic Usage
-----------
use package management system

- update package database first, or you may not find package because of wrong package version::
    
    su root
    pacman -Syy
    
    # download URL is based on package version.

- install basic tools for test::

    pacman -S sudo
    pacman -S git vim tmux
    pacman -S htop strace lsof

Use KVM on odroid c2
--------------------

1. Choose userspace tool:

   - build qemu/kvm-tool from scratch
   - use qemu/kvmtool build by VOS [1]
   - use Ubuntu's QEMU deb package
   
2. Choose guest OS image (Linux): :doc:`./guest_image` for more information.

   - Linaro Release by OpenEmbedded: minimal, LAMP
   - Ubuntu cloud image
   - Debian/Ubuntu/Fedora AArch64 release

[1] Virtual Open System

Misc
----

- QEMU networking: https://wiki.gentoo.org/wiki/QEMU/Linux_guest

Analysis difference of odroid-c2 / rpi3 ArchLinux ARM
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- ArchLinux default packages

  - odroid-c2 v.s. rpi3
  
    - kernel + bootloader
    - rpi3 has wireless repo
    - odroid-c2 has initramfs repo
