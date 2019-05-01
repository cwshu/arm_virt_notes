ivshmem
=======

- `Writing a Linux PCI Device Driver, A Tutorial with a QEMU Virtual Device <http://nairobi-embedded.org/linux_pci_device_driver.html>`_ 

現在情況: 

- QEMU/KVM 可使用 ivshmem, guest 上有兩種方法去 access ivshmem.

  - ARM64 會有 alignment access 的問題 (:doc:`/notes/misc/alignment_fault`)

- ivshmem 將 shared memory 放置於 PCI BAR2. BAR2 的 (start, len) 即為 shared memory 的 (start, len).

guest access ivshmem methods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
1. 用 ``/dev/mem`` 直接 hardcoded PCI BAR2 address, 可以 access 到 ivshmem

   - 取用 PCI BAR address:

     - qemu monitor: ``info pci`` 
     - guest linux kernel: ``lspci``

2. pci driver 使用 pci subsystem 的 API 獲得 BAR2 的 MMIO address, 然後封裝成一個 char device. 讓 userspace program 可以用 ``mmap()`` char device 來 access ivshmem.

目前測試環境
~~~~~~~~~~~~

1. host x86 pc, QEMU 2.9 + KVM + ivshmem legacy, guest x86 linux 4.4 (ubuntu 16.04)
2. host aarch64 board(odroid c2), QEMU 2.9 + KVM + ivshmem legacy, guest aarch64 linux 4.4 (ubuntu 16.04)

資源跟重現
~~~~~~~~~~
1. 準備 QEMU 2.9, ubuntu 16.04 的 qcow2 image (e.g. ``ubuntu_arm64.qcow2``)
2. 開一個 1M 大小的 POSIX shm: (e.g. ``/dev/shm/hello``)

   - 可參考: https://github.com/u1240976/testing_code/tree/master/posix_shm

3. run QEMU command with ivshmem::
    
    # ivshmem use 1M shm memory at /dev/shm/hello
    -device ivshmem,shm=hello,size=1
    
    # QEMU cmd (aarch64)
    $ qemu-system-aarch64 \
      -enable-kvm -m 1024 -cpu host -smp 4 -M virt -nographic \
      -bios ./QEMU_EFI.fd \
      -drive id=hd0,media=disk,if=none,format=qcow2,file=ubuntu_arm64.qcow2 \
      -device virtio-blk-device,drive=hd0 \
      -netdev user,id=net0,hostfwd=tcp:127.0.0.1:50022-:22 \
      -device virtio-net-pci,netdev=net0 \
      -device ivshmem,shm=hello,size=1 \
      -monitor telnet:127.0.0.1:60023,server,nowait

4. 測試 ivshmem 是否成功開啟

   - qemu monitor: ``monitor) info pci``
   - guest linux shell: ``$ lspci``, ``$ lspci -v -s 00:04.0``

5. access memory in guest OS

   - ``/dev/mem`` access: ``./ivshmem_access``
   - `pci driver access <https://github.com/u1240976/ivshmem_test>`_: guest/

Reference
---------
- `Writing a Linux PCI Device Driver, A Tutorial with a QEMU Virtual Device <http://nairobi-embedded.org/linux_pci_device_driver.html>`_ 
- `ivshmem use case 介紹 <http://www.linux-kvm.org/images/c/cc/2011-forum-nahanni.v5.for.public.pdf>`_

  - host/guest exchange file.
  - memcached: host OS's memcached pass through to multiple guest VM.
  - MPI

- `ivshmem 实现分析与性能测试 <http://blog.csdn.net/haitaoliang/article/details/22753423>`_
- `QEMU doc/ivshmem-spec.txt <https://github.com/qemu/qemu/blob/master/docs/specs/ivshmem-spec.txt>`_
