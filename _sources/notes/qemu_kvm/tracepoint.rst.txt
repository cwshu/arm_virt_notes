Linux Kernel Event Tracing in KVM
=================================

using frontend trace-cmd

prepare::

    $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/rostedt/trace-cmd.git trace-cmd
    $ make   # generate ./trace-cmd

    # KVM Tracepoints
    $ cat /sys/kernel/debug/tracing/available_events | grep kvm
    kvm:kvm_entry                   
    kvm:kvm_hypercall               
    kvm:kvm_hv_hypercall            
    kvm:kvm_pio                     
    kvm:kvm_fast_mmio  
    ...

    # source files
    include/trace/events/kvm.h
    arch/arm/kvm/trace.h

usage::
 
    $ sudo ./trace-cmd record -b 20000 -e kvm  # generate trace.dat
    $ ./trace-cmd report 

reference
---------

- kvm usage

  - http://www.linux-kvm.org/page/Tracing
  - http://people.cs.nctu.edu.tw/~chenwj/dokuwiki/doku.php?id=kvm

- [Doc] trace/events.txt: http://lxr.free-electrons.com/source/Documentation/trace/events.txt
- LWN - Using the TRACE_EVENT() macro (Part 1): https://lwn.net/Articles/379903/
