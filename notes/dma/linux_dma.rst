Linux DMA
=========

DMA Controller/Engine
---------------------

source code
~~~~~~~~~~~

- include/linux/dmaengine.h 
- drivers/dma/dmaengine.c
- include/linux/amba/pl330.h
- drivers/dma/pl330.c

document
~~~~~~~~

- `(maxime, 2015) An Overview of the DMAEngine Subsystem <http://events.linuxfoundation.org/sites/events/files/slides/ripard-dmaengine.pdf>`_ 
- ``Documentation/dmaengine/client.txt``
- ``Documentation/dmaengine/provider.txt``
- `ARM PL330 DMA Controller Manual <http://infocenter.arm.com/help/topic/com.arm.doc.ddi0424a/DDI0424A_dmac_pl330_r0p0_trm.pdf>`_ 
- 详解 ARM 的 AMBA 设备中的 DMA 设备 PL08X 的 Linux 驱动: http://www.crifan.com/files/doc/docbook/dma_pl08x_analysis/release/htmls/index.html

DMA Mapping
-----------

source code
~~~~~~~~~~~
- include/linux/dma-mapping.h
- arch/arm/mm/dma-mapping.c: implement 4 type of dma_map_ops

more: :doc:`/notes/dma/dma_mapping`

document
~~~~~~~~

- (Laurent, 2014) [ELC2014] `Mastering the DMA and IOMMU API <http://elinux.org/images/4/49/20140429-dma.pdf>`_ 
- `LDD3 ch15 Memory Mapping and DMA <http://www.makelinux.net/ldd3/?u=chp-15>`_
- ``Documentation/DMA-API-HOWTO.txt``

- ( 2011) [lwn] `ARM, DMA, and memory management <https://lwn.net/Articles/440221/>`_
- (Marek, Kyungmin, 2011, Samsung) [ELC Europe] `ARM DMA-Mapping Framework Redesign and IOMMU integration <http://elinux.org/images/7/7c/Elce11_szyprowski_park.pdf>`_

  - Three different implementations merged together (``arch/arm/mm/dma-mapping.c``)

    - linear non-coherent (most systems)
    - linear coherent (noMMU and Intel ixp23xx)
    - 'bounced' for systems with restricted or limited/broken DMA engines

- ( 2014) `ARM Dma-mapping Explained <http://linuxkernelhacker.blogspot.tw/2014/07/arm-dma-mapping-explained.html>`_
