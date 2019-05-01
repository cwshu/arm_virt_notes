KVM source code resource
========================

resource
--------
- `KVM源代码分析 1 <http://www.oenhan.com/kvm-src-1>`_

  - 5 篇系列文: basic, kvm_run, cpu, memory, IO
  - `QEMU monitor savevm loadvm 源代码分析 <http://www.oenhan.com/qemu-monitor-savevm-loadvm>`_

- `chenwj's wiki - KVM <http://people.cs.nctu.edu.tw/~chenwj/dokuwiki/doku.php?id=kvm>`_

  - cpu: VMCS, kvm, kvm_arch, kvm_vcpu, kvm_vcpu_arch, kvm_ioapic, kvm_lapic
  
    - kernel module init: vmx_init -> kvm_init -> kvm_arch_init

      - kvm_arch_hardware_setup, setup_vmcs_config, alloc_kvm_area

    - 設定 VMCS
    - running guest OS
    - emulate guest OS instruction

  - memory: shadow page table, EPT
  - IO
  - live migration
  - clock
  - QEMU

Misc
----
- `Stefan Hajnoczi - Open source and virtualization blog <http://blog.vmsplice.net>`_
- `RoyLuo's Notes - Blog索引 <http://royluo.org/2017/07/12/index/>`_
