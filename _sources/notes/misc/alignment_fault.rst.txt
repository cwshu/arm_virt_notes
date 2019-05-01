Alignment Fault
===============

terminology
-----------

- aligned memory access & unaligned memory access
  
  - `Linux Kernel Doc - Unaligned Memory Access <http://elixir.free-electrons.com/linux/v4.12.5/source/Documentation/unaligned-memory-access.txt>`_
  - `Linux Kernel Doc - arm/mem_alignment <http://elixir.free-electrons.com/linux/v4.12.5/source/Documentation/arm/mem_alignment>`_

- Alignment Fault: `ARM9 對 Alignment Fault 的定義 <http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0198e/Babcjcbe.html>`_

ARM CPU
-------

滿多 RISC 的 CPU 並不完整支援 unaligned memory access, 比如說 ARM9 所有的 memory access instruction 都不支援.

有一說是 ARMv5 (e.g. ARM9) 跟以前的 ARM CPU 不支援 unaligned memory access, ARMv6 以後 default 支援大部份的 aligned memory access:

- ``arch/arm/mm/alignment.c``: safe_usermode()::

     ARMv6 and later CPUs can perform unaligned accesses for
     most single load and store instructions up to word size.
     LDM, STM, LDRD and STRD still need to be handled.

- ``arch/arm/mm/alignment.c``: cpu_is_v6_unaligned()::

    return cpu_architecture() >= CPU_ARCH_ARMv6 && get_cr() & CR_U;

  - For ARMv6 and latter CPUs, it has CP15 c1 register U field. 
    which represent "Enables unaligned data access operations for mixed little-endian and big-endian operation".

    ref: `ARM11_CP15_c1 <http://infocenter.arm.com/help/topic/com.arm.doc.ddi0338g/Babgdhif.html>`_
  - ``arch/arm/include/asm/cp15.h``

ARMv8 的 normal memory 可以支援 unaligned memory access, 但 device memory 只支援 aligned memory access.

- `[Linux Kernel patch] pstore/ram: Use memcpy_toio instead of memcpy <https://patchwork.kernel.org/patch/9556649/>`_
- `ARM Cortex-A Series Programmer’s Guide for ARMv8-A - Ch5.1 Addressing <http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/ch05s01s02.html>`_

  - **Unaligned address support** 
    
      Except for exclusive and ordered accesses: 
      all loads and stores support the use of unaligned addresses when accessing normal memory.
      This simplifies porting code to A64.

- `Porting Linux to a new Architecture - ARMv8 AArch64 Software Bring-up <http://www.willdeacon.ukfsn.org/bitbucket/armv8/aci-presentation-02-13.pdf>`_ p.23


Compiler/Toolchain
------------------

我撞到的狀況

- ``strncpy()`` 跟 ``memset()`` 會碰到 alignment fault, ``memcpy()`` 似乎不會.

Detail

- strncpy() 的 len 為 5, 6, 7 或其他大於 8 但非 8 的倍數的常數時. 會碰到 alignment fault.

  - strncpy() len <= 8: gcc builtin strncpy()::

      strncpy(str, "hello world\n", 5);

      400be4:       f9401ba2        ldr     x2, [x29,#48]
      400be8:       90000000        adrp    x0, 400000 <_init-0x620>
      400bec:       9135e001        add     x1, x0, #0xd78
      400bf0:       aa0203e0        mov     x0, x2
      400bf4:       b9400022        ldr     w2, [x1]
      400bf8:       b9000002        str     w2, [x0]    # aligned write to x0[0:4]
      400bfc:       b8401021        ldur    w1, [x1,#1]
      400c00:       b8001001        stur    w1, [x0,#1] # unaligned write to x0[1:5]


  - strncpy() len > 8: `glibc strncpy() <https://github.com/bminor/glibc/blob/master/string/strncpy.c#L27>`_
  
    - 問題點似乎在 ``strncpy()`` 裡的 ``memset()``, 而且是 glibc aarch64 優化版的 ``memset()``
    - gdb 收到 SIGBUS 時在 strncpy() 的 memset 裏面 (``../sysdep/aarch64/memset.S``)
    - ``memset(str+13, '\0', 3);`` 可以重現 alignment fault.


reference
---------

- `ARM64 环境中的对齐异常 (alignment fault) 问题 <http://happyseeker.github.io/kernel/2016/02/26/alignment-fault-in-arm64.html>`_
  
  - memset 的 glibc 实现由于性能需要，使用了 DC ZVA 指令, 而 ARM64 架构中，DC ZVA 指令不能用于 device 类型的内存，否则就会触发 alignment fault 。
  - `arm64:lib: Use Linaro's memset routine to avoid DC instruction <http://lists.infradead.org/pipermail/linux-arm-kernel/2015-May/341675.html>`_

- `GCC builtin function <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>`_

  - ``-fno-builtin`` disable it.
