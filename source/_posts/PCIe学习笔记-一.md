---
title: PCIe学习笔记(一)
tags:
  - PCIe
description: >-
  本文是学习linux kernel中PCI子系统代码的一个笔记。PCI子系统的代码最主要的就是
  实现整个PCI树的枚举和资源的分配。本文先总体介绍，然后主要分析pci_create_root_bus
  函数，该函数实现pci_bus结构和pci_host_bridge结构的分配。本文分析的代码版本为内核 3.18-rc1
abbrlink: 4c075f83
date: 2021-07-11 23:48:09
categories:
---

 本文是学习linux kernel中PCI子系统代码的一个笔记。PCI子系统的代码最主要的就是
 实现整个PCI树的枚举和资源的分配。本文先总体介绍，然后主要分析pci_create_root_bus
 函数，该函数实现pci_bus结构和pci_host_bridge结构的分配。本文分析的代码版本为内核
 3.18-rc1

 pci_scan_root_bus()完成整个PCI树的枚举和资源分配。主要实现在下面的三个函数中：
    -->pci_create_root_bus()
    -->pci_scan_child_bus(b)
    -->pci_assign_unassigned_bus_resources(b)

 pci_create_root_bus 建立相应的数据结构pci_bus, pci_host_bridge等
 pci_scan_child_bus 枚举整个pci总线上的设备
 pci_assign_unassigned_bus_resources 分配总线资源

 下面直观的给出PCI(PCIe)硬件和软件的相互对应，也可以看出软件对硬件是怎么做抽象的。
 硬件结构:
 ```
		|   root bus: 0   ---->  struct pci_bus
		|
	+----------------+        ---->  struct pci_host_bridge
	| pcie root port |        ---->  struct pci_dev
	+----------------+        
		|   bus: 1        ---->  struct pci_bus
		|
	+----------------+
	| pcie net cards |        ---->  struct pci_dev
	+----------------+
 ```

先从硬件的角度说明PCIe总线系统的工作大致流程。PCIe总线系统是一个局部总线系统，
目的在于沟通内存和外设的存储空间。总结起来完成: 1. CPU访问外设的存储空间；2.
外设访问系统内存空间。一般介绍PCIe的时候说的RC，switch, EP，这里以arm Soc为例说明.
一般的, RC可以理解成PCIe host bridge, 有时也叫PCIe控制器，完成CPU域地址到PCI域
地址的转换，RC在Soc的内部。switch是一个独立的器件，和RC的接口相连，提供扩展。EP
是具有PCIe接口的网卡，SATA控制器等。PCI中还有一个概念是PCI桥, 实际的PCI桥存在
PCI总线中(不是PCIe总线), 完成系统的扩展，和host桥不同的是，PCI桥没有地址翻译的功能.
PCI总线中的switch每个端口相当于一个PCI桥(虚拟PCI桥), 完成PCI桥类似的功能。

每个PCI设备，包括host桥、PCI桥和PCI设备都有一个配置空间。PCIe中，这个配置空间的
大小是4K。CPU和外设根据配置空间的一些寄存器, 访问相应的地址。关键的寄存器有:
BAR, mem base/limit, I/O base/limit. mem base/limit, I/O base/limit指定当前PCI桥
mem空间和I/O空间的起始和大小, 在PCI桥中使用. BAR指示的是PCI设备的mem空间和I/O空间.
比如, 下图中PCIe net card的BAR空间(BAR指示的一段地址), 就可以存放网卡本身的寄存器.
一般情况, PCI桥的BAR是用不到的.
```
                    +----------------+ ----> PCIe host bridge
                    | pcie root port |
                    +----------------+ ----> in Soc
                            |
    --------------------------------------------------- ----> switch
    |                    +----------------+           |
    |                    |   pci bridge   |           |
    |                    +----------------+           |
    |                            |                    |
    |         -------------------------------         |
    |         |                             |         |
    | +----------------+           +----------------+ |
    | |   pci bridge   |           |   pci bridge   | |
    | +----------------+           +----------------+ |
    ---------------------------------------------------
              |
      +----------------+
      |  PCIe net card |
      +----------------+
```

整个pci枚举的过程最主要的就是配置pci桥和pci设备的BAR和mem、I/O base/limit
下面以此为主线分析整个pci枚举的过程。

在分析代码之前，先介绍基本的数据结构。代码围绕这些数据结构构成。
这几个核心的数据结构是: struct pci_bus, struct pci_dev, struct pci_host_bridge
每一个pci总线对应一个struct pci_bus结构，每一个pci设备(可以是host bridge,
普通pci device)对应一个pci_dev, 一个pci host bridge对应一个pci_host_bridge,
一般一个pci总线体系中有一个pci host bridge, 这一个pci总线系统也叫一个pci domain,
各个pci domain不可以直接相互访问。有些时候一个系统会有多个pcie root port, 这时
每个root port和下面的pci device组成各自的pci domain, 下面介绍各个结构中关键条目。
```
struct pci_bus:
        /* 指向该总线上游的pci桥的pci_dev结构 */
        struct pci_dev *self;
        /* 存储该总线的mem、I/O，prefetch mem等资源。由总线上游pci桥的
         * pci_dev结构中的resource中的第PCI_BRIDGE_RESOURCES到
         * PCI_BRIDGE_RESOURCES + PCI_BRIDGE_RESOURCE_NUM -1个元素复制得到
         * 发生于pci_scan_bridge()
         *           -->pci_add_new_bus()
         *              -->pci_alloc_child_bus()
         */
        struct resource *resource[PCI_BRIDGE_RESOURCE_NUM];
        struct list_head resources;

struct pci_host_bridge:
        /* 整个pci系统的mem, I/O资源作为一个个链表元素 */
        struct list_head windows;

struct pci_dev:
        /* 若该设备是pci桥，指向该桥的下游总线 */
        struct pci_bus *subordinate;
        /* 存该pci设备的BAR等资源, 在__pci_read_base()中初始化
         * pci_scan_device() 
         *    --> pci_setup_device() ...
         *        --> __pci_read_base
         * still do know where to init resource[PCI_BRIDGE_RESOURCES] ?
         */
        struct resource resource[DEVICE_COUNT_RESOURCE];
```
