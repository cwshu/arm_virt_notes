Platform Device, Device Tree, and Open Firmware
===============================================

APIs
----

- `platform_get_resouce 和 platform_get_irq 用法 <http://english0815.blogspot.tw/2016/05/platformgetresouce-platformgetirq.html>`_
- `A Tutorial on the Device Tree (Zynq) -- Part IV <http://xillybus.com/tutorials/device-tree-zynq-4>`_
- `蜗窝科技 - Device Tree（三）：代码分析 <http://www.wowotech.net/device_model/dt-code-analysis.html>`_

從 driver 獲得硬體 resource 的 API 來觀察.

- platform

  - platform_get_resource(): memory + irq
  - platform_get_irq(): irq

- of

  - of_address_to_resource()
  - of_irq_get()

從 device tree 獲得資料的進行, 有些是 kernel 開機初始化就 access 轉成 data structure [1], 
有些則是 lazy access, 在取用硬體資源前才去跟 device tree 要資料 [2].

[1] device tree in kernel init::

    __init customize_machine() => of_platform_populate() => of_dev_lookup() => of_address_to_resource()

[2] lazy access::


    platform_get_irq() => of_irq_get()

source code:

- driver/base/platform.c
- driver/of/*
- driver/of/platform.c

device tree format
------------------

- `Free Electron - Device Tree for Dummies <https://events.linuxfoundation.org/sites/events/files/slides/petazzoni-device-tree-dummies.pdf>`_
- `elinux - Device Tree Usage <https://elinux.org/Device_Tree_Usage>`_
- Documentation/devicetree/usage-model.txt 

notes
~~~~~

- ``compatible`` property: Every node represents a device have ``compatible`` property.

  - list of strings
  - first string: the exact device, format is ``"<manufacturer>,<model>"``
  - following strings: other devices that the device is compatible with

- node name: ``<name>[@<unit-address>]``
- address

  - ``reg``
  - parent's ``#address-cells`` and ``#size-cells``

- interrupt

  - ``interrupt-controller``: empty property. this device receive interrupt.
  - ``#interrupt-cells``, ``interrupts``: like ``#address-cells`` and ``reg``
  - ``interrupt-parent`` 
