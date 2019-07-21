---
title: Linux内核工程师是怎么步入内核殿堂的？
date: 2018-12-02 9:15:49
categories: Linux内核
tags:
- Linux内核
---

在知乎上看到了这个[问题](https://www.zhihu.com/question/304179651/answer/543389370)，借机总结一下自己在 Linux 内核学习研究上的经历和方法。

目前的工作实际上不是在搞 Linux 内核，但读大学的 4 年，其中有两年的时间在研究 Linux 内核和嵌入式 Linux。虽然已经好多年没再搞 Linux 内核，但上次项目需要，还是分析调试了 Android 的 low memory killer 驱动，并给 Android 的 binder 驱动增加了一些功能，对于 Linux 的内核的基本分析调试能力一直在。看到这个问题，分享一下自己的做法。

以我的理解，题主的问题可以分解为三个小问题，Linux 内核开发需要知道的基本背景知识，Linux 内核开发研究在技术上的路线图，以及 Linux 内核开发过程中的分析调试手法。
<!--more-->
Linux 内核开发需要的基本背景知识。为什么说是“基本”呢？主要是因为，如我们从操作系统原理一棵中所知道的那样，操作系统内核都是分为基本大的子系统的，如 I/O，任务调度，内存管理，文件系统和网络等等，这些子系统都是算法密集型的系统，不同子系统需要的背景知识还不太一样。

Linux 内核主要是用 C 语言写的，因而 C 语言开发的基本功必不可少。C 语言开发学习的经典著作很多，不仅有 [C程序设计语言](https://book.douban.com/subject/1139336/) 这样长生不衰的权威之作，还有，还有“C 语言开发的四书五经”和 C 语言开发——现代方法 这些非常好的作品。操作系统内核作为一个数据结构和算法密集型的领域，基本的数据结构和算法知识不可或缺，[数据结构](https://book.douban.com/subject/2024655/) 这本必修课自不必说，其它还有算法导论等等经典。

想要研究 Linux 内核，站在用户的角度来看待它，了解它的功能和接口是必须的。谁是 Linux 内核的用户，Linux 内核又为用户提供了什么功能，用户又是通过什么接口访问这些功能的呢？Linux 内核功能主要是通过系统调用暴露给用户的，系统调用通常又被封装为 C 库，需要使用操作系统内核功能的开发者通过 C 库使用 Linux 内核。相关领域非常经典的著作是 [UNIX环境高级编程（第3版）](https://book.douban.com/subject/25900403/)，读了这本书，应该就大体可以了解到，操作系统提供的大概就是文件操作，进程线程操作等功能了。[程序员的自我修养](https://book.douban.com/subject/3652388/) 也值得一看，可以了解一下程序加载运行的过程。

然后是操作系统原理和计算机体系结构。用户空间看到的操作系统功能和操作系统内部通常的设计结构并非严格对应的。通过操作系统原理，可以让我们看到操作系统内部通常的设计结构，如通常的子系统划分， I/O，任务调度，内存管理，文件系统和网络等，多个子系统的功能，都按照“一切皆文件”的 Unix 系统经典理念，通过文件有关的接口暴露给用户空间。操作系统原理有关的经典著作有斯托林斯的 [操作系统——精髓与设计原理](https://book.douban.com/subject/1506160/)，Andrew S·Tanenbaum 的 [现代操作系统](https://book.douban.com/subject/1390650/)和[操作系统设计与实现](https://book.douban.com/subject/2044818/)等，其中 [操作系统设计与实现](https://book.douban.com/subject/2044818/) 还是 Linus 当年设计实现 Linux 内核时的主要参考。[UNIX操作系统设计](https://book.douban.com/subject/1035710/) 和 [Linux内核设计与实现](https://book.douban.com/subject/1503819/) 也非常经典，值得一读，[Linux内核设计与实现](https://book.douban.com/subject/1503819/) 的作者是 Linux 内核的核心开发者，还是内核抢占子系统的作者。[自己动手写操作系统](https://book.douban.com/subject/1422377/) 也可以看一下。

Linux 内核研发是一个软硬件结合的领域，计算机组成原理和体系结构相关的知识必不可少，汇编语言，CPU 的 MMU 等等都需要有所了解。相关的经典著作有 [深入理解计算机系统](https://book.douban.com/subject/1230413/) 、[计算机组成与设计硬件/软件接口](https://book.douban.com/subject/2110638/)、[ARM嵌入式系统开发](https://book.douban.com/subject/1435663/) 和 [MIPS体系结构透视](https://book.douban.com/subject/3099520/) 等。答住过去关注的主要硬件平台是 ARM 的，因而关于 X86 平台的介绍，可以自行找涵盖了 MMU/保护模式 的 X86 汇编语言有关的著作来研究。

Linux 内核开发需要的基本背景知识大概就是这些。

Linux 内核开发研究在技术上的路线图。首先当然是从能够快速上手，能够快速带来成就感的地方开始。硬件平台可以用自己日常使用的 PC 机，装上 Linux 发行版，安装必要的编译链接等开发工具，差不多就可以上手了。Linux 内核的文档非常齐全，可以用 Git 把 Linux 内核的源码整个下载到本地，然后编译一下，熟悉熟悉 Linux 内核的构建系统，毕竟在整个的 Linux 内核研发过程中需要与这套系统打交道的地方很多。

就 Linux 内核研究和开发本身来说，最简单最方便的入手点就是驱动程序了。驱动程序可以从简单的字符设备内存型驱动开始，编写一个 “Hello world”字符设备驱动，暴露基本的文件操作，编译并动态加载进内核，在通过文件操作访问它时，可以让它仅仅打印一个“Hello world”。当然内核不能使用 C 语言标准库，内核打印的信息也不通过终端来看，而是通过 dmesg 命令来看。据说当前大多数的 Linux 内核研发工程师所做的工作，都是在搞驱动，驱动相关的代码在 Linux 内核代码中的占比远超一半。介绍 Linux 内核驱动的书很多，经典的长盛不衰但有点老的有 [Linux设备驱动程序](https://book.douban.com/subject/1723151/)，还有 [Linux设备驱动开发详解](https://book.douban.com/subject/2984156/) 和 [精通Linux驱动程序开发](https://book.douban.com/subject/3700970/) 等值得一读。实际的 Linux 内核驱动当然不可能像“Hello world”字符设备驱动那么简单。实际编写驱动时，硬件设备本身的协议需要仔细研读，诸如硬件设备有几个寄存器或者 IO 端口，对这些寄存器和 IO 端口的不同访问方式可以访问到什么功能等等。此外在 Linux 内核中运行的驱动程序代码，与内核中的其它子系统的交互几乎是必不可少的，因而 Linux 驱动相关的书读下来，会发现大部分都在讲内核中的内存分配和管理，Linux 内核中的并发同步方法等等。嵌入式 Linux 系统有关的书会讲许多关于硬件访问方法的内容，值得一读，如 [嵌入式Linux应用开发完全手册](https://book.douban.com/subject/3152027/)， [ARM嵌入式系统基础教程](https://book.douban.com/subject/3234368/)，[嵌入式Linux基础教程](https://book.douban.com/subject/4111412/)

很长时间没有通过读书来学习研究 Linux 内核，最近新出的书就不太了解了。不过在有了一定的基础之后，研究 Linux 内核最好的方法就是，阅读 Linux 内核官方相关模块的文档说明，阅读 Linux 内核的代码，然后做一些调试了。

研究学习 Linux 内核，除了从驱动入手外，也可以从另外两个点入手，一是内核的启动过程，二是系统调用。内核启动和系统调用也都是离用户最近的点，相对比较容易我们着手分析调试。

能够上手写一些 Linux 内核代码，并对 Linux 内核开发中能够用到的工具，如 Linux 内核提供的同步的函数，内存分配管理的函数等基础有所了解之后，就可以根据自己的喜好和项目需要，来具体深入学习研究特定子系统了，如任务调度，内存管理，文件系统和网络等等。

总结一下，Linux 内核开发研究在技术上的路线图大概是这样的：代码下载和构建 -> 选择从 Linux 驱动、内核的启动过程或系统调用三者之一入手开始自己的内核开发之旅 -> 学习 Linux 内核开发中能用到的工具和函数库 -> 根据工作和项目需要，选择某个自己感兴趣的子系统，如任务调度，内存管理，文件系统、IO、设备驱动和网络等等深入研究，完成自己的工作。

Linux 内核开发过程中的分析调试手法。硬件平台可以用自己的 PC 机，装上 Linux 发行版，然后在上面做实验。也可以用 ARM 平台做为硬件平台，可以买一块 ARM 开发板或者 Google 的 Nexus 系列手机。用 Google 的 Nexus 系列手机时，可以参考 Google 提供的官方文档来构建内核。不用硬件，用模拟器或虚拟机更方便，还不用担心把电脑或手机变成砖。Android 模拟器用来研究 Linux 内核非常方便。分析调试可以通过打断点或者打印日志来做。用 Android 模拟器来打内核的断点比较方便。

最后，[乐者为王](https://book.douban.com/subject/1395123/) 是一本介绍 Linus 的书，非常有意思。关于 Unix 社区的一些文化和哲学有关的内容可以经常看一下。