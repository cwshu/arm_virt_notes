STMicro ethernet driver note
============================

- `Linux Kernel - Documentation/networking/stmmac.txt <http://elixir.free-electrons.com/linux/latest/source/Documentation/networking/stmmac.txt>`_

Source File and Config
----------------------

``stmmac`` source code files introduction.

interface to upper layer:

  - stmmac_main.o: ``stmmac_netdev_ops``
  - stmmac_ethtool.o: ``stmmac_ethtool_ops``

internal operations:

  - dwmac: ``stmmac_ops``, ``stmmac_dma_ops``

    - dwmac_lib.o
    - dwmac1000: dwmac1000_core.o dwmac1000_dma.o 
    - dwmac100: dwmac100_core.o dwmac100_dma.o 

  - norm_desc.o, enh_desc.o: dma descriptor, ``stmmac_desc_ops``
  - ring_mode.o, chain_mode.o: ``stmmac_mode_ops``
  - other: stmmac_mdio.o, mmc_core.o, stmmac_hwtstamp.o, stmmac_ptp.o 

Kconfig options:

  - bus: ``stmmac_pci.c | stmmac_platform.c``
  
    - stmmac_platform.c (struct platform_driver stmmac_pltfr_driver)
  
  - dwmac: ``dwmac-meson.c | dwmac-sunxi.c | dwmac-sti.c``

header files

  - stmmac.h, common.h
  - descs.h, descs_com.h (dma descriptor)
  - dwmac100.h, dwmac1000.h, dwmac_dma.h
  - am_eth_reg.h, mmc.h, stmmac_ptp.h

Data Structure
--------------

4 ops in ``mac_device_info``::

  struct mac_device_info
  
  - stmmac_ops *mac;       // dwmac*.c
  - stmmac_dma_ops *dma;   // dwmac*.c
  - stmmac_desc_ops *desc; // *_desc.c
  - stmmac_mode_ops *mode; // *_mode.c

  e.g. priv->hw->mac->set_umac_addr() 
  use stmmac_ops dwmac1000_ops->set_umac_addr() at dwmac1000_core.c

``struct stmmac_priv``, which is stmmac's private_data of ``netdev``.::

    void __iomem* ioaddr;              // MMIO region base address
    struct mac_device_info* hw;        // 4 ops

    struct plat_stmmacenet_data* plat; // struct device->platform_data

Control Flow
------------

stmmac driver setting mac address
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

    // in stmmac_main.c
    stmmac_hw_setup()
    => priv->hw->mac->set_umac_addr() = dwmac1000_ops.set_umac_addr() = dwmac1000_set_umac_addr() 
    => stmmac_set_mac_addr()

stmmac MMIO region
~~~~~~~~~~~~~~~~~~

以下以 ``stmmac_set_mac_addr()`` 使用到的 MMIO region 為例子

1. MMIO region 透過 ``platform_get_resource()`` 得到, 並透過 ``ioremap()`` 轉成 virtual address 給 driver 使用.
::

     // driver use MMIO region by accessing priv->ioaddr
     stmmac_pltfr_probe(struct platform_device *pdev)
        
         res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
         addr = devm_ioremap_resource(dev, res);
         ...
         stmmac_dvr_probe(..., void __iomem *addr)
        
             priv->ioaddr = addr;

2. ``platform_get_resource()`` 透過 device tree 獲得 hard-coded 的 MMIO region address. ``platform_device`` 適用這套 API.
::

     // platform_get_resource() get resource from Device Tree
     // in arch/arm64/boot/dts/meson64_odroidc2.dts 

         ethmac: ethernet@0xc9410000{
              compatible = "amlogic, gxbb-rgmii-dwmac";
              reg = <0x0 0xc9410000 0x0 0x10000
              0x0 0xc8834540 0x0 0x8>
              ...
         }
         platform_get_resource(pdev, IORESOURCE_MEM, 0): start=0xc9410000, size=0x10000
         platform_get_resource(pdev, IORESOURCE_MEM, 1): start=0xc8834540, size=0x8

p.s. ``stmmac_set_mac_addr()`` 計算 MMIO offset 的方式
::

     // MMIO offset
     // in dwmac1000.h
     GMAC_ADDR_HIGH(reg) = 0x40 + reg*8 if reg <= 15;
     GMAC_ADDR_LOW(reg)  = 0x44 + reg*8 if reg <= 15;

net_device_ops/NAPI transfer and receive
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

driver transfer data to device.

``ndo_start_xmit()``

receive data from parameter ``sk_buff`` and transfer to ``dma_desc``?
  
- data::

    priv->cur_tx
    priv->dma_tx[entry].des2: dma_map_single(), skb_frag_dma_map()
    priv->tx_skbuff_dma[entry].buf: alias of priv->dma_tx[entry].des2
    priv->tx_skbuff[entry]

- functions::

    priv->hw->dma ops: enable_dma_transmission(priv->ioaddr);
    priv->hw->desc ops: prepare_tx_desc(), set_tx_owner(), close_tx_desc()
    skb_frag_dma_map(): dma_map_page() from ``sk_buff's`` page.

所以, 有幾個重點的 dependency 先處理

1. dma mapping API: dma_map_single(), dma_map_page()
2. sk_buff intro: 
3. stmmac hw datasheet? 我想看 TX/RX Ring 的 data structure

driver receive data from device.

``stmmac_poll()``

other net_device_ops
~~~~~~~~~~~~~~~~~~~~

- ndo_ioctl

  - stmmac_hwtstamp_ioctl()
  - phy_mii_ioctl(): forward to PHY ``ioctl()``

- ndo_open, ndo_stop
- ndo_poll_controller

- ndo_tx_timeout: stmmac_tx_err(priv);
- ndo_set_config: not supported (``-EOPNOTSUPP``)
- ndo_set_mac_address = NULL
- ndo_change_mtu: dev->mtu = new_mtu; netdev_update_features(dev);
- ndo_fix_features: check flags

- ndo_set_rx_mode: priv->hw->mac->set_filter(dev, priv->synopsys_id);

more than ``net_device_ops``, recieve need interrupt

- ``stmmac_interrupt()`` => ``stmmac_dma_interrupt()``: 應該是 interrupt handler

  - ``priv->hw->dma->dma_interrupt()`` 確認是否 handle_tx/rx, 或者 transmit error. 
    (handle_tx/rx 對應到 dwmac DMA 的 normal interrupt 的其中兩個種類)
  - 如果需要 handle_tx/rx, 則 trigger ``__napi_schedule()``

Other Info
----------

platform bus abstraction
~~~~~~~~~~~~~~~~~~~~~~~~

``stmmac_platform.c`` implement a platform driver ``stmmac_pltfr_driver``.
This driver is a wrapper of ``stmmac_driver``.

``stmmac_main.c`` registers this platform driver (``stmmac_pltfr_driver``) at kernel module initialization.

``stmmac_pltfr_driver`` implements platform driver ops (ops of ``struct platform_driver``), like probe, remove, driver.shutdown, driver.pm_ops.
``stmmac_pltfr_driver``'s ops depend on ``stmmac_driver`` ops, just do some customization.

customizations:

- MMIO region: ``platform_get_resource()``, ``ioremap()``
- 3 IRQs::

    priv->dev->irq = platform_get_irq_byname(pdev, "macirq");
    priv->wol_irq = platform_get_irq_byname(pdev, "eth_wake_irq");
    priv->lpi_irq = platform_get_irq_byname(pdev, "eth_lpi");

- ``struct device``'s platform_data

  - ``stmmac_probe_config_dt()``

    - ``platdata_copy_from_machine_data(device, plat);``
    - ``setup_mac_addr(pdev, mac);``

  - ``plat_stmmacenet_data->setup()``
  - ``plat_stmmacenet_data->init()``

dwmac-meson.c
~~~~~~~~~~~~~

implements 2 functions ``setup()`` and ``fix_mac_speed()``

setup functions to device's platform_data ``plat_stmmacenet_data``::

    platdata_copy_from_machine_data(const struct of_device_id *device, struct plat_stmmacenet_data *plat){
        if (device->data) {
            const struct stmmac_of_data *data = device->data;
            plat->setup = data->setup;
            plat->fix_mac_speed = data->fix_mac_speed;
            ...
        }
    }

usage about these functions::

    1. priv->plat->fix_mac_speed()
    2. custom setup function in platform_driver initialization.

    stmmac_pltfr_probe() => stmmac_probe_config_dt() => platdata_copy_from_machine_data()
        struct platform_device *pdev;
        struct plat_stmmacenet_data *plat_dat = dev_get_platdata(&pdev->dev);
        ...
        // custom setup function
        plat_dat->setup()

dwmac DMA
~~~~~~~~~

- source codes: dwmac_lib.c, dwmac_dma.h
- DMA registers
  
  - CSR: control and status registers.
  - ``dwmac_dma.h``: DMA CSR Mapping(MMIO), from ``DMA_BUS_MODE(0x1000)`` to ``DMA_HW_FEATURE(0x1058)``. (CSR0 to CSR8, and ... CSR?)
  - CSR1: enable dma transmission
  - CSR7: mask interrupt and record interrupt type. (``DMA_INTR_ENA_*``)
  - DMA status register (CSR5): ``DMA_STATUS_*``
  - DMA control register (CSR6): ``DMA_CONTROL_*``

- DMA interrupt: ``DMA_INTR_ENA_*``

  - normal interrupt: 5 kinds, from NIE to ERE
  - abnormal interrupt: 10 kinds, from AIE to TSE
