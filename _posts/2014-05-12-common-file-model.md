---
layout:    post
title:     通用文件模型
category:  虚拟文件系统
description: 通用文件模型...
tags: 文件模型
---
VFS所隐含的主要思想在于引入了一个通用文件模型（*common file model*），这个模型能够表示所有支持的文件系统。该模型严格反映传统Unix文件系统提供的文件模型。这非常强大，因为Linux希望以最小的额外开销运行它本地文件系统。不过，要实现每个具体的文件系统，必须将其物理组织结构转换为虚拟文件系统通用文件模型。

例如，在通用文件模型中，每个目录被看作一个文件，可以包含若干文件和其他的子目录。但是，存在几个非Unix的基于磁盘的文件系统，它们利用文件分配表（*File Allocation Table，FAT*）存放每个文件在目录树中的位置，在这些文件系统中，存放的是目录而不是文件。为了符合VFS的通用文件模型，对上述基于FAT的文件系统的实现，Linux必须在必要的时候能够快速建立对应于目录的文件。这样的文件只作为内核内存的对象而存在。

从本质上来说，Linux内核不能对一个特定的函数进行硬编码来执行注入*read()*或*loctl()*这样的操作，而是对每个操作都必须使用一个指针，指向要访问的具体文件系统的适当函数。

我们可以把通用文件模型看作是面向对象的，在这里，对象是一个软件结构，其中既定义了数据结构也定义了其上的操作方法。处于效率的考虑，Linux的编码并未采用面向对象的程序设计语言。因此对象作为普通的C数据结构来实现，数据机构中指向函数的字段就对应于对象的方法。通用文件模型由下列对象类型组成：

**超级块对象（*superblock object*）**

存放已安装文件系统的有关信息，对基于磁盘的文件系统，这类对象通常对应于存放在磁盘上的文件系统控制块（*filesystem control block*）。

**索引节点对象（*inode object*）**

存放于具体文件的一般信息，对基于磁盘的文件系统，这类对象通常对应于存放在磁盘上的文件控制块（*file control block*）。每个索引节点对象都有一个索引节点号。这个节点号唯一地标识文件系统中的文件。

**文件对象（*file object*）**

存放打开文件与进程之间进行交互的有关信息，这类信息仅当进程访问文件期间存在于内核内存中。

**目录项对象（*dentry object*）**

存放目录项，也就是文件的特定名称，与对应文件进行链接的有关信息。每个磁盘文件系统都以自己特有的方式将改类信息存在磁盘上。

{:.center}
![vfs](/blog/images/vfs2.png){:style="max-width:550px"}

{:.center}
进程与VFS对象之间的交互

如上图所示的一个简单的示例，说明进程怎样与文件系统交互。三个不同的进程已经打开同一个文件，其中两个进程使用同一个硬链接。在这种情况下，其中的每一个进程都能使用自己的文件对象，但只需要两个目录项对象，每个硬链接对应一个目录项对象。这两个目录项对象指向同一个索引节点对象，该索引节点对象标识超级块对象，以及随后的普通磁盘文件。

VFS除了能为所有文件系统的实现提供一个通用接口外，还具有另一个与系统性能相关的重要作用。最近最常使用的目录项对象被放在所谓目录项高速缓存的磁盘高速缓存中，以加速从文件路径名到最后一个路径分量的索引节点的转换过程。

一般来说，磁盘高速缓存（*disk cache*）属于软件机制，它允许内核将原本存在磁盘上的某些信息保存在RAM中，以便对这些数据的进一步访问能快速进行而不必慢速访问磁盘本身。

磁盘高速缓存不同于硬件高速缓存或内存高速缓存，后两者都与磁盘或其他设备无关，硬件高速缓存是一个快速静态RAM，它加快了直接对慢速动态RAM的请求。内存高速缓存是一种软件机制，引入它是为了绕过内核分配器。可参考[硬件高速缓存和TLB](/blog/posts/system-hardware-cache-and-tlb/)章节。

除了目录项高速缓存和索引节点高速缓存之外，Linux还能使用其他磁盘高速缓存，其中最为重要的一种就是所谓的页高速缓存。