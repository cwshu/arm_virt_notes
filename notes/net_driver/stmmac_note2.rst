STMicro ethernet driver note 2
==============================

Misc
----

- stmmac_main.c

  - net_device_ops.ndo_open() => stmmac_open()
  - => alloc_dma_desc_resources()
  - => stmmac_hw_setup()

    - ``init_dma_desc_rings(struct net_device *dev)``: 

      - set priv->dma_buf_sz by priv->hw->mode->set_16kib_bfsize(dev->mtu)
      - foreach priv->dma_rx: stmmac_init_rx_buffers() => __netdev_alloc_skb() + dma_map_single()
      - foreach priv->dma_tx: zeroing(init) priv->tx_skbuff_dma[i]?? lazy allocation??
      - stmmac_clear_descriptors(): priv->hw->desc->init_rx_desc()

    - ``stmmac_init_dma_engine(struct stmmac_priv *priv) => priv->hw->dma->init()``
    - [mac] priv->hw->mac->set_umac_addr(priv->ioaddr, dev->dev_addr, 0);
    - [bus] priv->plat->bus_setup(priv->ioaddr);
    - priv->hw->mac->core_init(priv->ioaddr, dev->mtu);
    - stmmac_set_mac(priv->ioaddr, true): Enable the MAC Rx/Tx
    - stmmac_dma_operation_mode(priv);
    - stmmac_mmc_setup(priv);
    - ``stmmac_eee_init(struct stmmac_priv *priv)``
    
      - phy_init_eee(), priv->hw->mac->set_eee_timer()
      - init_timer(), add_timer(), priv->hw->mac->set_eee_timer()
      - priv->hw->mac->set_eee_pls()

