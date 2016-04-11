---
layout: post
title:  Linux 虚拟化和 PCI 透传技术(转)
date:   2013-07-15 12:53
categories: Web
tags: [KVM,透传]
---


处理器已经演变为针对虚拟环境提高性能，但 I/O 方面发生了什么变化呢？了解一种名为设备（或 PCI）透传（passthrough）的 I/O 性能增强技术，这种创新技术通过使用来自 Intel® (VT-d) 或 AMD (IOMMU) 的硬件支持改进 PCI 设备的性能。

平台虚拟化是在两个或多个操作系统之间共享一个平台，以便更有效地利用资源。但<em>平台</em> 并不只是意味着一个以上的处理器，它还包含组成平台的其他重要元素，比如存储器、网络和其他硬件资源。某些硬件资源可以轻松虚拟化，比如处理器和存储器；而另一些硬件资源则不然，比如视频适配器和串口。当共享不可能或没用时，Peripheral Component Interconnect (PCI) 透传技术提供有效使用这些资源的方法。本文探索<em>透传（passthrough）</em>技术的概念及其在管理程序（hypervisor）中的实现，详细介绍支持这个最新创新技术的管理程序。
<h2><a name="emulation"></a>平台设备模拟</h2>
在探索透传技术之前，让我们先讨论一下如今设备模拟在两个管理程序架构中是如何工作的。第一个架构将设备模拟整合到管理程序中，而第二个架构将设备模拟推到管理程序之外的一个应用程序中。

<em>管理程序中的设备模拟</em> 是在 VMware 工作站产品（一个基于操作系统的管理程序）中实现的一个公共方法。在这个模型中，管理程序包含各种客户操作系统能够共享的公共设备，如虚拟磁盘、虚拟网络适配器和其他必需的平台元素。这个特定模型如图 1 所示。
<a name="fig1"></a><b>图 1. 基于管理程序的设备模拟</b>
<img src="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/figure1.gif" alt="基于管理程序的设备模拟" width="361" height="228" />

第二个架构称为<em>用户空间设备模拟</em>（见图 2）。顾名思义，这种设备模拟是在用户空间中实现的，而不嵌入到管理程序中。QEMU（不仅提供设备模拟，还提供一个管理程序）提供设备模拟，用于大量独立管理程序，如 Kernel-based Virtual Machine (KVM) 和 VirtualBox 等。这个模型更具优势，因为设备模拟独立于管理程序，因而可以在多个管理程序之间共享。另外，这个模型还支持任意设备模拟，无须管理程序（以特权状态运行）负担这个功能。
<a name="fig2"></a><b>图 2. 用户空间设备模拟</b>
<img src="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/figure2.gif" alt="用户空间设备模拟" width="437" height="228" />

将设备模拟从管理程序推向用户空间有一些明显的优势，最大的优势涉及所谓的<em>可信计算基础（trusted computing base，TCB）</em>。一个系统的 TCB 是对该系统安全性很关键的所有安全组件的集合。有一点是显而易见的：如果系统被最小化，出现 bug 的可能性也就更小，因此系统也就越安全。这个原理也适用管理程序。管理程序的安全性很重要，因为它分隔多个独立的客户操作系统。管理程序中的代码越少（将设备模拟推到特权较低的用户空间中），将特权泄露给不可信用户的机率也就越少。

基于管理程序的设备模拟的另一个变体是准虚拟化（paravirtualized）驱动程序。在这个模型中，管理程序包含物理驱动程序，每个客户操作系统包含一个管理程序可以感知的驱动程序，这个驱动程序与管理程序驱动程序（称为<em>准虚拟化</em> 或 <em>PV</em> 驱动程序）配合工作。

无论设备模拟发生在管理程序内还是在一个客户虚拟机（VM）之上，模拟方法都是相似的。设备模拟能够模拟一个特定设备（如 Novell NE1000 网络适配器）或一个特定磁盘类型（如 Integrated Device Electronics [IDE]）。物理硬盘可以完全不同 — 例如，尽管一个 IDE 驱动器被模拟为客户操作系统，物理硬件平台可以使用一个串口 ATA (SATA) 驱动器。这种技术很有用，因为 IDE 支持在许多操作系统中都很普遍，可以用作一个通用标准，而不是要求所有操作系统都支持更高级的驱动器类型。
<div></div>
<a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#ibm-pcon"> </a>
<h2><a name="device_passthrough"></a>设备透传技术</h2>
正如上面介绍的两个设备模型所示，设备共享是有代价的。无论设别模拟是在管理程序还是在一个独立 VM 中的用户空间中执行，都存在开销。只要有多个客户操作系统需要共享这些设备，这个开销就是值得的。如果共享不是必须的，则有更有效的方法来共享这些设备。

因此，在最高层面上，设备透传就是向一个特定客户操作系统提供一种设备隔离，以便该设备能够被那个客户操作系统独占使用（见图 3）。但这种技术为什么有用？设备透传之所以有价值，原因有很多，其中两个最重要的原因是性能以及提供本质上不能共享的设备的专用权。
<a name="fig3"></a><b>图 3. 管理程序内的设备透传</b>
<img src="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/figure3.gif" alt="管理程序内的设备透传" width="437" height="228" />

对于性能而言，使用设备透传可以获得近乎本机的性能。对于某些网络应用程序（或那些拥有高磁盘 I/O 的应用程序）来说，这种技术简直是完美的。这些网络应用程序没有采用虚拟化，原因是穿过管理程序（达到管理程序中的驱动程序或从管理程序到用户空间模拟）会导致竞争和性能降低。但是，当这些设备不能被共享时，也可以将它们分配到特定的客户机中。例如，如果一个系统包含多个视频适配器，则那些适配器可以被传递到特定的客户域中。

最后，可能有一些只有一个客户域使用的专用 PCE 设备，或者有一些不受管理程序支持因而应该被传递到客户机的设备。单独的 USB 端口可以与一个给定域隔离，一个串口（自身不是可共享的）可以与一个特定客户机隔离。
<div></div>
<a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#ibm-pcon"> </a>
<h2><a name="covers"></a>设备模拟背后的秘密</h2>
早期的设备模拟类型在管理程序中实现影子（shadow）形式的设备接口，以便为客户操作系统提供一个到硬件的虚拟接口。这个虚拟接口包含预期的接口，包括表示设备（如 shadow PCI）的虚拟地址空间和虚拟中断。但是，由于有一个设备驱动程序与虚拟接口通信，且有一个管理程序为实际硬件转换这种通信，因此开销非常大 — 特别是在诸如网络适配器之类的高带宽设备中。

Xen 使 PV 方法（上一小节介绍过）得以流行，PV 方法通过使客户操作系统驱动程序意识到它正在被虚拟化来减少性能降低幅度。在本例中，客户操作系统将不会看到一个设备（比如网络适配器）的 PCI 空间，而是一个提供高级抽象（比如包接口）的网络适配器应用程序编程接口（API）。这种方法的缺点是客户操作系统必须针对 PV 进行修改，优点是在某些情况下您可以得到近乎本机的性能。

在设备透传技术早期发展过程中，开发人员使用一个瘦模拟模型，在该模型中，管理程序提供基于软件的内存管理（将客户操作系统地址空间转换为可信主机地址空间）。尽管开发人员在早期提供了隔离一个设备和一个客户操作系统的方法，但那种方法缺乏大型虚拟化环境需要的性能和伸缩性。幸运的是，处理器供应商已经为下一代处理器装备了一些指令，以支持管理程序和用于设备透传的逻辑，包括终端虚拟化和直接内存访问（DMA）支持。因此，新的处理器提供 DMA 地址转换和权限检查以实现有效的设备透传，而不是捕获并模拟对管理程序下的物理设备的访问。
<h2><a name="hardware_support"></a>设备透传的硬件支持</h2>
Intel 和 AMD 都在它们的新一代处理器架构中提供对设备透传的支持（以及辅助管理程序的新指令）。Intel 将这种支持称为<em>Virtualization Technology for Directed I/O</em> (VT-d)，而 AMD 称之为 <em>I/O Memory Management Unit</em> (IOMMU)。不管是哪种情况，新的 CPU 都提供将 PCI 物理地址映射到客户虚拟系统的方法。当这种映射发生时，硬件将负责访问（和保护），客户操作系统在使用该设备时，就仿佛它不是一个虚拟系统一样。除了将客户机映射到物理内存外，新的架构还提供隔离机制，以便预先阻止其他客户机（或管理程序）访问该内存。Intel 和 AMD CPU 提供更多虚拟化功能，您可以在 <a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#resources">参考资料</a> 部分了解更多信息。

另一种帮助将中断缩放为大量 VM 的技术革新称为 <em>Message Signaled Interrupts</em> (MSI)。MSI 将中断转换为更容易虚拟化的消息（缩放为数千个独立中断），而不是依赖将被关联到一个客户机的物理中断 pin。从 PCI 2.2 开始，MSI 就已经可用，但 PCI Express (PCIe) 也提供 MSI，在 PCIe 中，MSI 支持将结构缩放为多个设备。MSI 是理想的 I/O 虚拟化技术，因为它支持多个中断源的隔离（而不是必须通过软件多路传输或路由的物理 pin）。
<div></div>
<a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#ibm-pcon"> </a>
<h2><a name="hypervisor_support"></a>设备透传的管理程序支持</h2>
使用最新的支持虚拟化的处理器架构，有多个管理程序和虚拟化解决方案支持设备透传。您将在 Xen 和 KVM 以及其他管理程序中发现设备透传支持（使用 VT-d 或 IOMMU）。在多数情况下，客户操作系统（域为 0）必须被编译为支持透传，这通常作为一个内核构建时选项提供。也许还需要对主机 VM 隐藏设备（Xen 中使用 <code>pciback</code> 实现）。PCI 中有一些限制（例如，一个 PCIe-to-PCI 桥接器后面的 PCI 设备必须被分配到相同的域），但 PCIe 没有这种限制。

另外，您将在 libvirt（以及 virsh）中发现设备透传的配置支持，这为底层管理程序使用的配置模式提供一个抽象。
<div></div>
<a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#ibm-pcon"> </a>
<h2><a name="problems"></a>设备透传问题</h2>
设备透传带来的一个问题体现在实时迁移方面。<em>实时迁移</em> 是指一个 VM 在迁移到一个新的物理主机期间暂停迁移，然后又继续迁移，该 VM 在这个时间点上重新启动。实时迁移是在一个物理主机网络上支持负载平衡的一个很好的特性，但使用透传设备时它会产生问题。PCI 热插拔（有几个关于它的规范）就是需要解决的一个问题。PCI 热插拔允许 PCI 设备从一个给定内核进出，这很理想 — 特别是将 VM 迁移到新主机上的管理程序时（设备需要在这个新管理程序中拔出然后再插入）。当设备被模拟（比如虚拟网络适配器）时，模拟提供一个抽象层以抽象物理硬件。这样，一个虚拟网络适配器可以在该 VM 内轻松迁移（这个 VM 还得到 Linux® 绑定驱动程序的支持，该驱动程序支持将多个逻辑网络适配器绑定到相同的接口上）。
<div></div>
<a href="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/index.html#ibm-pcon"> </a>
<h2><a name="next_steps"></a>I/O 虚拟化的未来</h2>
I/O 虚拟化的未来实际上已经在今天实现。例如，PCIe 包含虚拟化支持。一种适合服务器虚拟化的虚拟化概念被称为 <em>Single-Root I/O Virtualization</em> (SR-IOV)，这种虚拟化技术（通过 PCI-Special Interest Group 或 PCI-SIG 创建）在单根复杂实例（在本例中为一个带有多个 VM 的服务器，这些 VM 共享一个设备）中提供设备虚拟化。另一个变体（称为 <em>Multi-Root IOV</em>）支持大型拓扑（比如刀片服务器，其中多个服务器能够访问一个或多个 PCIe 设备）。从某种意义上说，这种技术支持任意规模的大型设备网络，该网络可以包含服务器、终端设备和交换机（用于设备发现和包路由）。

通过 SR-IOV，一个 PCIe 设备不仅可以导出多个 PCI 物理功能，还可以导出共享该 I/O 设备上的资源的一组虚拟功能。这个简化的服务器虚拟化架构如图 4 所示。在这个模型中，不需要任何透传，因为虚拟化在终端设备上发生，从而允许管理程序简单地将虚拟功能映射到 VM 上以实现本机设备性能和隔离安全。
<a name="fig4"></a><b>图 4. 通过 SR-IOV 实现透传</b>
<img src="http://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/figure4.gif" alt="通过 SR-IOV 实现透传" width="361" height="288" />