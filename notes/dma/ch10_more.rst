Interrupt Handling More
=======================

``SA_SHIRQ`` and IRQ internal
-----------------------------

- struct irq_desc, irqaction
- https://gist.github.com/u1240976/e4358c1ba6dda64e8f41a74bf7689ab

- irq_desc->depth implements nested call of ``disable_irq()/enable_irq()``

``disable_irq()`` v.s. ``local_irq_disable()``
----------------------------------------------

internal:

- ``local_irq_*()`` use CPU instructions to disable CPU receive IRQ.
- ``disable_irq()`` use irqchip feature to disable irqchip send IRQ.

``disable_irq()``::

    kernel/irq/manage.c
    kernel/irq/chip.c

    disable_irq()/disable_irq_nosync() => irq_disable() => irq_chip.irq_disable()

``local_irq_*()`` compatibility in ARM
--------------------------------------

``local_irq_*()`` in ARM

    source file::
    
        include/linux/irqflags.h
        arch/arm/include/asm/irqflags.h
        arch/arm64/include/asm/irqflags.h
    
    各平台實作基於 ``arch_local_irq_save()``.
    
    很多不同平台的 ARM instructions support.
    
    enable/disable::

        1. cpsid / cpsie instructions
        2. CPSR, CPSR_c registers
        3. daifset / daifclr registers

    save/restore::
        
        1. primask register
        2. CPSR_c register
        3. daif register
        
Some Topics/Questions

- How to write a safe interrupt handler?
- trace ``/proc/interrupt`` and ``/proc/stat``.

- ``disable_irq()`` sync problem. why ``disable_irq()`` calls ``__disable_irq_nosync()`` before ``synchronize_irq()``?

Reference
---------

- http://kernel.meizu.com/linux-interrupt.html
- http://kernel.meizu.com/linux-workqueue.html
- http://www.wowotech.net/irq_subsystem/gic_driver.htm

