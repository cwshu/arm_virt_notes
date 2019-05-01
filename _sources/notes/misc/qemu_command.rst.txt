收集一些 ARM guest 時使用的 QEMU Commands
=========================================

我用 ubuntu server 在 odroid 的測試::

    qemu-system-aarch64 \
        -enable-kvm -cpu host \       # [kvm/tcg]
        -M virt -bios QEMU_EFI.fd \   # ARM 環境大多需要
        -m 1024 -smp 4 \              # 資源分配, 不影響相容性. ram 太少會跑很慢.
        -nographic \                  # 圖像, -std vga 在 arm 上好像不太 work. 可以試試看 vnc
        -drive id=cdrom1,media=cdrom,if=none,file=../images/ubuntu-16.04.1-server-arm64.iso \
        -device virtio-scsi-device -device scsi-cd,drive=cdrom1 \
        -drive id=hd0,media=disk,if=none,format=qcow2,file=ubuntu_arm64.qcow2 \
        -device virtio-blk-device,drive=hd0 \
        -netdev user,id=user -device virtio-net-device,netdev=user

odroid_c2 wiki kvm example::

    qemu-system-aarch64 \
        -enable-kvm -cpu host \
        -M virt -bios QEMU_EFI.fd \
        -smp 2 -m 1024 \
        -nographic \
        -drive if=none,id=image,file=xenial-server-cloudimg-arm64-uefi1.img \
        -device virtio-blk-device,drive=image \
        -drive if=none,id=cloud,file=cloud.img \
        -device virtio-blk-device,drive=cloud \
        -netdev user,id=user -device virtio-net-device,netdev=user

buildroot::

    qemu-system-aarch64 \
        -cpu cortex-a57 \
        -machine virt -machine type=virt \
        -smp 1 -m 2048 \
        -nographic \
        -kernel aarch64-linux-3.15rc2-buildroot.img  --append "console=ttyAMA0"

    # access local file-system
    qemu-system-aarch64 \
        -cpu cortex-a57 \
        -machine virt -machine type=virt \
        -smp 1 -m 2048 \
        -nographic \
        -kernel aarch64-linux-3.15rc2-buildroot.img --append "console=ttyAMA0" \
        -fsdev local,id=r,path=/home/alex/lsrc/qemu/rootfs/trusty-core,security_model=none \
        -device virtio-9p-device,fsdev=r,mount_tag=r

別人用 qemu-system-aarch64 emulate, rootfs 用 archlinux arm generic::

    qemu-system-aarch64 \
        -cpu cortex-a57 \
        -M virt -bios QEMU_EFI.fd \
        -m 2048 -smp 1 \
        -serial stdio \
        -kernel mnt/boot/Image -initrd mnt/boot/initramfs-linux-fallback.img \
        -append "root=/dev/vda1" \
        -drive if=none,file=disk.img,id=hd0 \
        -device virtio-blk-device,drive=hd0 

        # extend
        -initrd mnt/boot/initramfs-linux.img
        -drive format=raw
        -netdev user,id=unet -device virtio-net-device,netdev=unet

debootstrap::

    qemu-system-aarch64 \
        -cpu cortex-a57 \
        -machine type=virt \
        -smp 1 -m 8192 \
        -nographic \
        -kernel /srv/chroots/vmlinuz-3.13.0-34-generic -initrd /srv/chroots/initrd.img-3.13.0-34-generic \
        --append "root=/dev/vda1 rw console=ttyAMA0 --" \
        -drive file=/srv/chroots/trusty.qcow2,if=none,id=blk \
        -device virtio-blk-device,drive=blk \ 
        -netdev tap,id=net0 \
        -device virtio-net-device,netdev=net0,mac=00:00:00:00:00:00 \

Note
----
1. ARM guest 很需要 ``-M virt -bios QEMU_EFI.fd``, ``QEMU_EFI.fd`` 來自 Linaro
2. ARM guest 的 vga 情況可能跟 x86 完全不同, 目前只有 ``-nographic`` 可以正常顯示圖像
   
   - ``-serial stdio`` 可以正常顯示 kernel log 沒問題.

3. cdrom 用 virtio-scsi-device + scsi-cd, disk(img, qcow2) 用 virtio-blk-device
4. virtio-net-device
