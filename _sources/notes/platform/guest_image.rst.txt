Prepare AArch64 Linux guest OS image
====================================

1. Linaro image

  - http://releases.linaro.org/openembedded/aarch64/16.07/
  
    - kernel: Image
    - rootfs1 (minimal): linaro-image-minimal-genericarmv8-20160724-830.rootfs.tar.gz
    - rootfs2 (LAMP): linaro-image-lamp-genericarmv8-20160724-830.rootfs.tar.gz

  - usage

    - `busybox udhcpc <http://felix-lin.com/linux/busybox-%E6%87%89%E7%94%A8-udhcpc/>`_
    - linaro-minimal has udhcpc default script.

  - more about Linaro OpenEmbedded

    - `Bootstrapping ARM 64-bit with OpenEmbedded <http://elinux.org/images/6/61/2012-ELCE-Bootstrapping-ARM-64bit-with-OpenEmbedded.pdf>`_
    - `Booting Linaro ARMv8 OE images with Qemu <http://suihkulokki.blogspot.tw/2014/08/booting-linaro-armv8-oe-images-with-qemu.html>`_
    - https://wiki.linaro.org/Cycles/1509/Release

2. Ubuntu cloud image

   - `odroid-c2 wiki <http://odroid.com/dokuwiki/doku.php?id=en:c2_ubuntu_cloud>`_
   - `Booting ubuntu 16.04 cloud images on Arm64 <http://suihkulokki.blogspot.tw/2016/05/booting-ubuntu-1604-cloud-images-on.html>`_
   - problem: ubuntu cloud image 增大::

       qemu-img resize <image>+5G
     
     - https://gist.github.com/larsks/3933980

3. Some distro provides AArch64 ISO

  - Ubuntu 16.04: http://cdimage.ubuntu.com/releases/16.04/release/
  - Debian 8.6: http://cdimage.debian.org/debian-cd/8.6.0/arm64/iso-cd/
  - Fedora 23: https://dl.fedoraproject.org/pub/fedora-secondary/releases/23/

Problems in AArch64 ISO installed rootfs
----------------------------------------

ubuntu, debian, fedora 的 AArch64 rootfs 皆無法在 odroid_c2 kvm 上跑起來.

- problems

  - ubuntu server iso 找不到 rootfs, 無法安裝
  - debian 安裝 grub 會失敗
  - fedora 裝完之後, boot 只能進到 EFI firmware

- solution

  - CD-ROM mount: use virtio-scsi-device and scsi-cd for CD-ROM
  - UEFI: :ref:`boot_uefi_issue`

debian failed:

  .. image:: pic/debian_failed.png

fedora failed:

  .. image:: pic/fedora_failed.png

Misc
----

qemu & rootfs in arm64 資源
~~~~~~~~~~~~~~~~~~~~~~~~~~~
- odroid c2 wiki

  - qemu mode: kvm
  - rootfs: ubuntu cloud image
  - 相似: https://wiki.ubuntu.com/ARM64/QEMU

- https://www.bennee.com/~alex/blog/2014/05/09/running-linux-in-qemus-aarch64-system-emulation-mode/

  - qemu mode: tcg
  - rootfs: buildroot, Linaro
  - http://stenliao.blogspot.tw/2014/10/run-aarch64-linux-on-qemu.html

- http://osmanov-dev-notes.blogspot.tw/2016/03/arch-linux-armv8-vm-on-gentoo-amd64.html

  - qemu mode: tcg
  - rootfs: ArchlinuxARM generic. 用 loop mount + dump rootfs 來做 image. 
    但 image 好像保持 loop mount, qemu 會用 -kernel 直接指定 image 裡的 kernel.

- https://gist.github.com/ecliptik/81ad7484d522097dca7f

  - qemu mode: tcg
  - rootfs: ubuntu or debian from debootstrap

- https://fedoraproject.org/wiki/Architectures/AArch64/Install_with_QEMU

  - qemu mode: tcg
  - rootfs: fedora 23 (aarch64 server)

稍微整理一下

- rootfs 種類

  - buildroot
  - ubuntu cloud image
  - ubuntu server iso
  - ubuntu from debootstrap
  - debian from debootstrap
  - fedora server iso
  - archlinux arm generic rootfs (pacstrap)

- buildroot: 建立 embedded system rootfs 的 tool, 相似於 yocto/OpenEmbedded.
- QEMU command 分析: ../misc/qemu_command.rst

