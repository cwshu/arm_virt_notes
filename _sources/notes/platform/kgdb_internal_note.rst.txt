KGDB internal note
==================

- kgdboc: kgdb over console

kgdboc init
-----------

從 kgdboc 設定 sysfs 碰到 error 開始 ::

    $ echo 'ttyS0' > /sys/module/kgdboc/parameters/kgdboc
    -bash: echo: write error: No such device

這個 sysfs entry 是一個 kernel module kgdboc 的 parameter.

kgdboc parameter 帶有 setter, 也就是說當設定 sysfs parameter 時會呼叫 setter.
kgdboc 的 setter 會呼叫 ``configure_kgdboc()``, 把 parameter 中輸入的 tty driver 設定到 kgdboc 之中.

``configure_kgdboc()`` 有幾個核心: (假設 input parameter 為 ``ttyS0``)

- 參數如果有 ``kbd`` 或 ``kdb``, 使用 ``register_kbd()`` 去設定 ``kdb_poll_funcs``
- ``tty_find_polling_driver()``: 在 tty_drivers 當中搜尋 ``ttyS0`` 並回傳. ``(p.s. tty_driver="ttyS" and tty_line="0")``
- 搜尋 console_drivers 當中有無跟該 tty_driver 同名的 driver, 有的話設定 ``kgdboc_io_ops.is_console = true``.
- 將 ``kgdb_tty_driver`` 設定為 ``ttyS0``, 該 driver 為 ``struct kgdb_io kgdboc_io_ops`` 的 backend.
- 最後呼叫 ``kgdb_register_io_module()`` 跟 ``kgdb_register_nmi_console()`` 進行註冊.
  ``kgdb_register_io_module()`` 會將 kgdboc 設定為 KGDB 的 IO driver.

在 ``configure_kgdboc()`` 當中, ``kgdb_register_io_module()`` 之前碰到的錯誤幾乎都會回傳 ``NODEV``.

不過這次的問題是出在 ``tty_find_polling_driver()`` 的檢查上:

    當 kgdboc 使用的 tty_driver 是以 uart 為 backend 時, uart_driver 需要支援 polled mode.
    具體來說是 uart_driver 需要實作 uart_ops->poll_put_char() 跟 poll_get_char() 兩個函式. 
    
    trace::

        tty_find_polling_driver() => struct tty_operations->poll_init()

        in tty_driver serial_core:
          tty_operations->poll_init() == uart_poll_init()

        uart_poll_init(): check if struct uart_ops->poll_get_char() and ->poll_put_char() is exist
          => calls custom init function uart_ops->poll_init() if it exist.

        # odroid c2/amlogic S905
        uart_driver meson_uart doesn't implement poll_get_char() and poll_put_char().

相關文件

- `kgdboc internals <https://www.kernel.org/pub/linux/kernel/people/jwessel/kdb/kgdbocDesign.html>`_
- `consoles: polling support, kgdboc <https://lkml.org/lkml/2008/2/10/23>`_

分檔:

- KGDB core and arch support

  - kernel/debug/debug_core.c: kgdb_register_io_module(), struct kgdb_io dbg_io_ops;
  - arch/arm64/kernel/kgdb.c: struct kgdb_arch arch_kgdb_ops;

- KGDB over console: one implementation of KGDB IO driver

  - drivers/tty/serial/kgdboc.c: implementation of struct kgdb_io kgdboc_io_ops
  - drivers/tty/serial/kgdb_nmi.c: kgdb_register_nmi_console()

- TTY driver/subsystem

  - drivers/tty/tty_io.c: tty_find_polling_driver(), tty_drivers
  - include/linux/tty_driver.h: struct tty_driver, tty_operations;

- UART driver/subsystem

  - drivers/tty/serial/serial_core.c: struct tty_operations uart_ops (same meaning as tty_driver serial_core)
  - include/linux/serial_core.h: struct uart_ops, uart_port, uart_driver;
  - fs/proc/proc_tty.c: ``/proc/tty/drivers``, ``proc/tty/driver/<tty_driver>``

- include/linux/console.h: struct console
- kernel/printk/printk.c: console_drivers

KGDB
----

KGDB::

  include/linux/kgdb.h
  kernel/debug/{debug_core.c,gdbstub.c}

  struct kgdb_arch, kgdb_io
  arch_kgdb_ops
  dbg_io_ops, kgdb_register_io_module()

  kgdb_breakpoint() => arch_kgdb_breakpoint()
  kgdb_handle_exception()
  dbg_{set,get}_reg()

``debug_core.c``:

  有個全域的 struct kgdb_io 變數 dbg_io_ops,
  kgdb_register_io_module() 便是把一個 kgdb_io 的實作 assign 給 dbg_io_ops.

- kgdb_register_nmi_console() check

``kgdboc.c``:

  其中一套 struct kgdb_io 的實作, kgdboc_io_ops.
  kgdboc_io_ops 的實作使用全域的 struct tty_driver 變數 kgdb_tty_driver.
  get/put_char() 的運算會去使用 kgdb_tty_driver 下面的 tty_operations.

TTY and console
---------------

global variable ``tty_drivers``, ``/proc/tty/drivers``, char devices ``/dev/tty*``


``/proc/tty/drivers``: list all ``struct tty_driver`` in ``(global var) tty_drivers``.::

    5 columns: driver_name, name, major, minor_start~minor_end, type
    
    # major and minor are char device number, like /dev/ttyS0 or /dev/ttyUSB0
    # 主要有 4 個 type: system, serial, pty, console.
    # 每個 type 下還有 subtype, 比如說 pty 就有分 pty master/slave.

``/proc/tty/driver/<tty_driver>``::

    show uart_driver->uart_state->uart_port

    /proc/tty/driver/meson_uart
    
    many columns: line, uart, mmio, irq, tx, rx, ..., flags

    # line: uport_line
    # uart: uart_type

tty and uart

    uart subsystem 有寫一個 tty_driver serial_core, 該 tty_driver 可以使用 uart subsystem 下的所有 driver 來實作 tty_driver.

UART
----

note

- struct ``uart_ops`` and ``uart_port``
- https://www.kernel.org/doc/Documentation/serial/driver

UART driver is also like network driver. It has send and recieve function, move their data by DMA, recieve data when interrupt trigger them.

- send function: uart_ops.start_tx()
- recieve function: interrupt handler => ... => dma_rx() 

examples:

- ``drivers/tty/serial/8250/``
