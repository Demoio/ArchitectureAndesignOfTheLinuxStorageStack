# 一切的开始-- 虚拟文件系统

即使在软件开发领域取得天文般的进步，linux 内核仍然保留着非常复杂的代码。开发者或者维护人员还有一切内核的高手不断进入内核添加一些特性。其他的爱好者尝试理解他并解决一些困惑。

自然地，关于Linux及其内部机制已有大量文献，从一般的系统管理到内核编程。

几十年来，已经出版了数百本书籍，涵盖了众多重要的操作系统主题，例如进程创建、线程、内存管理、虚拟化、文件系统实现以及CPU调度。这本数聚焦linux存储技术栈和他的多层次结构。

我们将开始介绍VFS，他在进程访问文件系统数据起关键作用。由于我们打算在本书中从上到下全面介绍整个存储堆栈，因此深入理解虚拟文件系统（Virtual Filesystem）至关重要，因为它是内核中I/O请求的起点。我们将介绍用户空间和内核空间的概念，理解系统调用，并了解Linux中“一切皆文件”的理念是如何与虚拟文件系统相关联的。

在这一章节中 我们将涵盖下面的内容：
1. 了解现代数据中心的存储技术
2. 系统调用
3. 了解虚拟文件系统的必要
4. VFS概览
5. 解释“一切皆文件”哲学

### 技术要求

在继续之前，我认为有必要在此指出，某些技术主题可能比其他主题更难让初学者理解。由于本节的目标是理解Linux内核及其主要子系统的内部工作原理，因此具备对操作系统概念（尤其是Linux）的基本理解将非常有帮助。最重要的是，要以耐心、好奇心和乐于学习的态度来面对这些主题。本章中介绍的命令和示例是与发行版无关的，可以在任何Linux操作系统上运行，例如Debian、Ubuntu、Red Hat和Fedora。文中有一些对内核源代码的引用。如果您想下载内核源代码，可以从https://www.kernel.org下载。与本章相关的操作系统软件包可以按以下方式安装：

For Ubuntu/Debian:
- sudo apt install strace
- sudo apt install bcc

For Fedora/CentOS/Red Hat-based systems:
- sudo yum install strace
- sudo yum install bcc-tools


## 了解现代数据中心的存储技术

> 在数据尚未获得之前就进行理论推导，这是个严重的错误。不知不觉中，人就会开始歪曲事实以适应理论，而不是根据事实来调整理论。——阿瑟·柯南·道尔爵士

计算、存储、网络 时构建任何基础设置的必要模块。你的应用程序表现如何，通常取决于这三层的综合性能。现代数据中心中运行的工作负载从流媒体服务到机器学习应用各不相同。

随着云计算平台的迅速崛起和广泛采用，所有基本的构建模块现在都对终端用户进行了抽象。当你的应用程序变得需要更多资源时，增加更多的硬件资源已成为常态。为了迁移到更强大的硬件平台上，通常会跳过排查性能问题的步骤。

在三大基础组件——计算、存储和网络中，存储通常在大多数情况下被视为瓶颈。对于数据库等应用来说，底层存储的性能至关重要。当基础设施承载着关键任务和对时间敏感的应用程序（如在线事务处理OLTP）时，存储的性能经常被忽视。在响应I/O请求时出现的最小延迟都可能影响应用程序的整体响应性能。

衡量存储性能最常用的指标是延迟。存储设备的响应时间通常以毫秒为单位进行测量。将其与你常见的处理器或内存相比，这些部件的响应时间是以纳秒为单位进行测量的，你就会明白存储层的性能如何影响整个系统的运行。这就导致了应用需求与底层存储实际能够提供的性能之间出现不一致的情况。在过去的几年中，现代存储驱动器的大部分进步都集中在容量方面。然而，存储硬件的性能提升并没有以相同的速度发展。与计算功能相比，存储性能显得相形见绌。出于这些原因，存储常常被称为数据中心的“三腿狗”（即存在明显短板）。  

在谈及存储介质的选择时，需要注意的是，无论硬件多么强大，其功能始终存在局限性。应用程序和操作系统根据硬件进行调整同样非常重要。对应用程序、操作系统和文件系统参数进行细致调优，可以显著提升整体性能。为了充分发挥底层硬件的潜力，I/O层次结构中的所有层级都需要高效运行。


### 在linux中与存储进行交互

linux内核在用户空间和内核空间有清晰的差异，所有的硬件资源，如CPU、内存和存储，都位于内核空间。对于任何用户空间的资源来说 如果想要访问内核空间的资源，必须通过系统调用如图Figure1.1。

![alt text](image.png)

用户空间 指的是所有位于内核之外的应用程序和进程。内核空间包括设备驱动程序等程序，这些程序可以不受限制地访问底层硬件。用户空间可以视为一种沙箱机制，用来限制终端用户程序修改关键的内核功能。

用户空间和内核空间的概念深深的植入到现代应用程序的设计之中。传统的x86 CPU 使用称为“环（rings）”的保护域概念来共享和限制对硬件资源的访问。处理器提供4个rings或者模式，序号从0到3.现代处理器设计只在其中两个模式下运行，即环 0 和环 3。用户空间应用程序运行在环 3 中，具有有限的内核资源访问权限。内核则占据环 0。内核代码在这里执行，并与底层硬件资源进行交互。

当进程需要读取或写入文件时，它们需要与物理磁盘上方的文件系统结构进行交互。每种文件系统使用不同的方法来组织磁盘上的数据。进程的请求不会直接到达文件系统或物理磁盘。为了让物理磁盘服务于进程的 I/O 请求，必须经过内核中整个存储层级。该层级中的第一层称为 虚拟文件系统。以下的图 图 1.2 突出了虚拟文件系统的主要组件：

![alt text](image-1.png)

Linux 中的存储栈由多个紧密相连的层组成，这些层共同确保通过统一的接口抽象了对物理存储介质的访问。在接下来的内容中，我们将基于这一结构进行扩展，增加更多的层次。我们将尽力深入探讨每一层，了解它们如何协同工作。

本章将专注于虚拟文件系统及其各种特性。在接下来的章节中，我们将解释并揭示 Linux 中更常用的文件系统的一些底层工作原理。然而，考虑到这里将多次提到 文件系统 这一词汇，我认为有必要简要地对不同的文件系统类型进行分类，以避免任何混淆：

- 块级文件系统：块级或基于磁盘的文件系统是存储用户数据的最常见方式。作为普通操作系统用户，这些就是用户大多数交互的文件系统。像扩展文件系统版本 2/3/4（Ext 2/3/4）、扩展文件系统（XFS）、Btrfs、FAT 和 NTFS 等文件系统都被归类为基于磁盘或块级文件系统。这些文件系统以块为单位。块大小是文件系统的一个属性，只有在创建设备上的文件系统时才能设置。块大小表示文件系统在读写数据时使用的大小。我们可以将其视为文件系统中存储分配和检索的逻辑单位。一个可以按块访问的设备因此被称为块设备。任何连接到计算机的存储设备，无论是硬盘还是外部 USB，都可以归类为块设备。传统上，块级文件系统是挂载在单个主机上的，不允许多个主机之间共享。

- 集群文件系统：集群文件系统也是块级文件系统，使用基于块的访问方法来读写数据。不同之处在于，它们允许单个文件系统被多个主机同时挂载和使用。集群文件系统基于共享存储的概念，这意味着多个主机可以同时访问同一个块设备。Linux 中常用的集群文件系统包括 Red Hat 的全球文件系统 2（GFS2）和Oracle 集群文件系统（OCFS）。

- 网络文件系统（NFS）：NFS 是一种允许远程文件共享的协议。与常规的块级文件系统不同，NFS 基于多个主机之间共享数据的概念。NFS 的工作模式是客户端和服务器的概念。后端存储由 NFS 服务器提供。挂载 NFS 文件系统的主机系统称为客户端。客户端和服务器之间的连接是通过常规以太网实现的。所有 NFS 客户端共享 NFS 服务器上的单一文件副本。NFS 的性能不如块级文件系统，但它仍然广泛应用于企业环境，主要用于存储长期备份和共享常用数据。

- 伪文件系统 /proc (procfs)和/sys (sysfs)属于这一类别。这些目录包含虚拟或临时文件，其中包含有关不同内核子系统的信息。这些伪文件系统也是虚拟文件系统的一部分，正如我们将在万物皆文件章节中看到的那样。

现在我们对用户空间、内核空间和不同类型的文件系统有了基本的了解，接下来我们将解释应用程序如何通过系统调用在内核空间中请求资源。


## 理解系统调用

当在浏览将解释虚拟文件系统和应用程序交互的图片时，你也许注意到在用户空间和虚拟文件系统之间有一个中介层。这层被称为系统调用层。为了请求一些内核中的服务，应用将调用这层系统调用层。这些系统调用提供了终端用户应用程序访问内核空间资源像是处理器，内存，存储的途径。

系统调用层主要有以下三个目的：
- 确保安全： 系统调用层防止应用直接修改内核空间的资源。
- 抽象：  应用不必关系底层硬件的规格
- 可移植性：用户程序可以在所有实现相同接口集的内核上正确运行

通常在系统调用和应用程序接口（api）之间有一些困惑。一个API是给一个程序用的接口组合。API是一组程序使用的编程接口。这些接口定义了两个组件之间通信的方法。API是在用户空间中实现的，并说明如何获得特定的服务。系统调用是一种更底层的机制，它使用中断来向内核发出明确的请求。在Linux中，系统调用接口由标准C库提供。

如果调用进程生成的系统调用成功，将返回一个文件描述符。文件描述符是一个用于访问文件的整数编号。例如，当使用 open() 系统调用打开一个文件时，会将一个文件描述符返回给调用进程。一旦文件被打开，程序便使用文件描述符对文件进行操作。所有读取、写入和其他操作都是通过文件描述符进行的。

每个进程至少会打开三个文件——标准输入、标准输出和标准错误——分别由文件描述符0、1和2表示。接下来打开的文件将被分配文件描述符值3。如果我们通过ls进行一些文件列表操作，并运行一个简单的strace，open系统调用将返回值3，该文件描述符代表的是文件——在本例中是/etc/hosts。之后，这个文件描述符值3会被fstat和close调用来执行进一步的操作：


```
strace ls /etc/hosts
root@linuxbox:~# strace ls /etc/hosts
execve("/bin/ls", ["ls", "/etc/hosts"], 0x7ffdee289b48 /* 22 vars */)
= 0
brk(NULL) = 0x562b97fc6000
access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT (No such file or
directory)
access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=140454, ...}) = 0
mmap(NULL, 140454, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fbaa2519000
close(3) = 0
access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT (No such file or
directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_
CLOEXEC) = 3
```

[其余代码为简洁起见省略。]

在x86系统上，大约有330个系统调用。这个数字在其他架构上可能会有所不同。
每个系统调用都由一个唯一的整数编号表示。你可以使用ausyscall命令列出你系统上的可用系统调用。这将列出系统调用及其对应的整数值：

```
ausyscall –dump
root@linuxbox:~# ausyscall --dump
Using x86_64 syscall table:
0 read
1 write
2 open
3 close
4 stat
5 fstat
6 lstat
7 poll
8 lseek
9 mmap
10 mprotect
```

[其余代码为简洁起见省略。]

```
root@linuxbox:~# ausyscall --dump|wc -l
334
root@linuxbox:~#
```

下面是一些常见的系统调用：
```
System call Description
open (), close () Open and close files
create () Create a file
chroot () Change the root directory
mount (), umount () Mount and unmount filesystems
lseek () Change the pointer position in a file
read (), write () Read and write in a file
stat (), fstat () Get a file status
statfs (), fstatfs () Get filesystem statistics
execve () Execute the program referred to by pathname
access () Checks whether the calling process can access the file pathname
mmap () Creates a new mapping in the virtual address space of the calling process
```

那么，系统调用在与文件系统交互中扮演什么角色呢？正如我们在接下来的章节中将看到的那样，当用户空间的进程生成一个系统调用来访问内核空间中的资源时，它首先会与虚拟文件系统（VFS）进行交互。这个系统调用首先由内核中相应的系统调用处理程序处理，在验证了所请求的操作之后，处理程序会调用VFS层中的适当函数。VFS层将请求传递给适当的文件系统驱动模块，该模块会在文件上执行实际的操作。

我们需要理解其中的原因——为什么进程要与虚拟文件系统交互，而不是直接与磁盘上的实际文件系统交互？在接下来的章节中，我们将尝试弄清楚这一点。

总之，在Linux中，系统调用接口实现了通用的方法，这些方法可以被用户0空间的应用程序用来访问内核空间中的资源。

## 文件系统的必要性

一个标准的文件系统是一组数据结构，它们决定了用户数据如何在磁盘上进行组织。最终用户可以通过常规的文件访问方法与这个标准文件系统进行交互，并执行常见任务。每一个linux系统都提供至少一个文件系统，当然他们都声称比自己比其他的更好更快更安全。主流的linux发行版默认使用xfs或者ext4文件系统。这些文件系统具有多种功能，并且被认为在日常使用中稳定可靠。

然而，linux没有限制只支持这两种文件系统。使用linux的好处是他提供多个文件系统的支持，他们都被认为是ext4户=或者xfs的完美替代方案。因此linux可以和其他操作系统共存。其他常见的文件系统有：ext4的旧版本ext2,ext3.还有Btrfs, ReiserFS, OpenZFS, FAT, 和 NTFS。

在使用多个分区时，用户可以从众多可用的文件系统中进行选择，并根据需要在每个磁盘分区上创建不同的文件系统。


物理硬盘中可寻址的最小单元是一个扇区。对于文件系统 最小的单位是块。一个块可以被认为是一组连续的扇区。文件系统的所有操作都是按照块进行的。文件系统的所有操作都是以块为单位进行的。不同的文件系统以多种方式来寻址和组织这些块。每个文件系统用不同的数据结构来组织磁盘块上的数据。在每个存储分区上使用不同的文件系统可能会难以管理。鉴于Linux支持的文件系统种类繁多，试想一下，如果应用程序需要了解每一个文件系统的具体细节，会是多么复杂的事情。为了兼容某个文件系统，应用程序必须为它所使用的每个文件系统实现一种独特的访问方法。这将使应用程序的设计变得几乎不切实际。

抽象接口在Linux内核中起着至关重要的作用。在Linux中，无论使用的是哪种文件系统，最终用户或应用程序都可以通过统一的访问方式与文件系统进行交互。这一切都是通过虚拟文件系统层实现的，该层通过一个全面的接口隐藏了具体的文件系统实现。


## VFS概览

为了确保应用程序在面对不同的文件系统时不会有什么障碍，Linux内核在终端用户应用程序和用于存储数据的文件系统之间实现了一个层次结构。这层被叫做VFS，这个VFS不是一个标准的文件系统，向是ext4和xfs。因此，有些人更喜欢使用“虚拟文件系统切换”这个术语。

想想《纳尼亚传奇》中的魔法衣橱。这个衣橱实际上是通往纳尼亚魔法世界的门户。一旦你穿过衣橱，就可以探索这个新世界并与那里的居民互动。衣橱有助于进入魔法世界。同样地，VFS（虚拟文件系统）为不同的文件系统提供了一个入口。

VFS 定义了一个通用接口，使得多个文件系统能够在 Linux 中共存。值得再次强调的是，VFS 并非指标准的基于块的文件系统。我们在谈论的是一个抽象层，它提供了最终用户应用程序与实际块文件系统之间的联系。通过 VFS 中的标准化，应用程序可以执行读写操作，而无需担心底层的文件系统。

如下图，VFS 位于用户空间程序与实际文件系统之间
![alt text](image-2.png)

为了使 VFS 能够为双方提供服务，以下条件必须成立：

- 所有最终用户应用程序需要根据 VFS 提供的标准接口来定义它们的文件系统操作
- 每个文件系统需要提供 VFS 提供的通用接口的实现。

我们已经解释过，用户空间中的应用程序在需要访问内核空间中的资源时，需要生成系统调用。通过 VFS 提供的抽象，像read()和write()这样的系统调用可以正常工作，而不管使用的是哪种文件系统。这些系统调用能够跨文件系统边界工作。我们不需要特别的机制将数据移到不同或非本地文件系统。例如，我们可以轻松地将数据从 Ext4 文件系统移动到 XFS，反之亦然。从高层次来说，当进程发出read()或write()系统调用以读取或写入文件时，VFS 会搜索要使用的文件系统驱动程序，并将这些系统调用转发给该驱动程序。

## 通过vfs实现通用的文件系统接口

VFS 的主要目标是以最小的开销在内核中表示多种文件系统。当进程请求对文件进行读写操作时，内核将其替换为该文件所在文件系统的特定函数。为了实现这一点，每个文件系统必须根据 VFS 进行适配。

为了更好地理解，我们来看看以下示例。

考虑linux中的copy命令，假设我们将一个文件从ext4拷贝到xfs中。这个复制操作是如何完成的？cp 命令如何与两个文件系统进行交互？

![alt text](image-3.png)

首先，cp命令并不关心所使用的文件系统。我们已经将 VFS 定义为实现抽象的层。因此，cp命令无需关心文件系统的细节。它将通过标准的系统调用接口与 VFS 层进行交互。具体来说，它将发出open()和read()系统调用来打开和读取要复制的文件。一个已打开的文件在内核中由文件数据结构表示（正如我们将在下一章第二章中学习的那样，解释 VFS 中的数据结构）。

当cp生成这些通用系统调用时，内核将通过指针将这些调用重定向到文件所在文件系统的适当函数。为了将文件复制到 XFS 文件系统，write()系统调用会传递给 VFS。然后，这个调用会再次被重定向到实现该功能的 XFS 文件系统的特定函数。通过向 VFS 发出的系统调用，cp进程可以使用 Ext4 的read()方法和 XFS 的write()方法执行复制操作。就像一个开关，VFS 将在它们指定的文件系统实现之间切换通用的文件访问方法。

内核中的读、写或任何其他功能没有默认定义——这也是虚拟一词的含义。对于这些操作的解释取决于底层文件系统。就像利用 VFS 提供的抽象的用户程序一样，文件系统也能从这种方法中受益。文件系统不需要重新实现文件的常见访问方法。

这还挺酷的，对吧？但是如果我们想从 Ext4 复制一些东西到一个非原生文件系统呢？像 Ext4、XFS 和 Btrfs 这样的文件系统是专门为 Linux 设计的。如果其中一个文件系统是 FAT 或 NTFS，情况会怎样呢？

不得不承认，VFS 的设计偏向于源自 Linux 系列的文件系统。对于最终用户来说，文件和目录之间有明确的区别。在 Linux 的哲学中，一切都是文件，包括目录。原生 Linux 文件系统，如 Ext4 和 XFS，都是在考虑到这些细微差别的情况下设计的。由于实现上的差异，非原生文件系统如 FAT 和 NTFS 并不支持所有 VFS 操作。Linux 中的 VFS 使用 inode、超级块和目录项等结构来表示文件系统的通用视图。非原生 Linux 文件系统并不使用这些结构。那么，Linux 是如何适配这些文件系统的呢？以 FAT 文件系统为例，FAT 文件系统来源于不同的世界，并未使用这些结构来表示文件和目录。它不会将目录视为文件。那么，VFS 是如何与 FAT 文件系统交互的呢？

内核中与文件系统相关的所有操作都与 VFS 数据结构紧密集成。为了支持 Linux 上的非原生文件系统，内核动态构建相应的数据结构。例如，为了满足 FAT 等文件系统的通用文件模型，将在内存中动态创建对应于目录的文件。这些文件是虚拟的，只存在于内存中。这是一个需要理解的重要概念。在原生文件系统中，像 inode 和超级块这样的结构不仅存在于内存中，还存储在物理介质上。相反，非 Linux 文件系统只需要在内存中执行这些结构的实现。

### 查看源码

如果我们查看内核 的源码，fs提供的代码都在fs/目录下。所有以 .c 结尾的源文件都包含了不同 VFS 方法的实现。子目录包含特定文件系统的实现。

![alt text](image-4.png)

你会注意到open.c 和read_write.c文件，当在用户空间调用open()和read()或write()时，内核会调用这些文件。这些文件包含了太多的代码，所以我们不会在这里编写代码，仅仅做一个小练习。然而，这些文件中有一些重要的代码片段突出了我们之前所解释的内容。让我们快速看一下读取和写入函数。

SYSCALL_DEFINE3宏是定义系统调用的标准方式，其中一个参数就是系统调用的名称。

为了写一个系统调用，看下面的函数，注意到其中一个参数是文件描述符。

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
size_t, count)
{
 return ksys_write(fd, buf, count);
}
```

这是read的系统调用。
```C
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t,
count)
{
 return ksys_read(fd, buf, count);
}
```

两者都调用了ksys_write () 和 ksys_read () 的系统调用实现。
```C
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
 struct fd f = fdget_pos(fd);
 ssize_t ret = -EBADF;
******* Skipped *******
 ret = vfs_read(f.file, buf, count, ppos);
******* Skipped *******
 return ret;
}

ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t
count)
{
 struct fd f = fdget_pos(fd);
 ssize_t ret = -EBADF;
******* Skipped *******
 ret = vfs_write(f.file, buf, count, ppos);
 ******* Skipped *******
 return ret;
}
```

vfs_read () 和 vfs_write () 函数的存在表明我们正在过渡到 VFS。这些函数查找底层文件系统的 file_operations 结构，并调用适当的 read () 和 write () 方法：
```C
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
******* Skipped *******
        if (file->f_op->read)
                ret = file->f_op->read(file, buf, count, pos);
        else if (file->f_op->read_iter)
                ret = new_sync_read(file, buf, count, pos);
******* Skipped *******
return ret;
}
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
******* Skipped *******
        if (file->f_op->write)
                ret = file->f_op->write(file, buf, count, pos);
        else if (file->f_op->write_iter)
                ret = new_sync_write(file, buf, count, pos);
 ******* Skipped *******
       return ret;
}

```

每个文件系统定义了支持操作的 file_operations 结构体。内核源代码中有多个 file_operations 结构体的定义，每个文件系统都有独特的定义。该结构体中定义的操作描述了读写函数如何执行：
```C
root@linuxbox:/linux-5.19.9/fs# grep -R "struct file_operations" * | wc -l
453
root@linuxbox:/linux-5.19.9/fs# grep -R "struct file_operations" *
9p/vfs_dir.c:const struct file_operations v9fs_dir_operations = {
9p/vfs_dir.c:const struct file_operations v9fs_dir_operations_dotl = {
9p/v9fs_vfs.h:extern const struct file_operations v9fs_file_operations;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_file_operations_dotl;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_dir_operations;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_dir_operations_dotl;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_cached_file_operations;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_cached_file_operations_dotl;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_mmap_file_operations;
9p/v9fs_vfs.h:extern const struct file_operations v9fs_mmap_file_operations_dotl;
```
[其余代码因简洁性省略。]

file_operations 结构体适用于各种文件类型，包括常规文件、目录、设备文件和网络套接字。一般来说，任何可以使用标准文件 I/O 操作打开和操作的文件类型，都可以通过此结构体来处理。

### 跟踪 VFS 函数

Linux 提供了相当多的跟踪机制，可以让我们窥见其底层工作原理。其中之一是 funccount。顾名思义，funccount (`sudo dnf install bpftrace,export PATH=$PATH:/usr/share/bcc/tools`)用于计算函数调用的次数：
```C
root@linuxbox:~# funccount --help
usage: funccount [-h] [-p PID] [-i INTERVAL] [-d DURATION] [-T] [-r] [-D]
                 [-c CPU]
                 pattern
Count functions, tracepoints, and USDT probes

```

为了测试和验证我们之前的理解，我们将在后台运行一个简单的复制进程，并使用 funccount 程序跟踪由于执行 cp 命令而调用的 VFS 函数。由于我们只需要计算 cp 进程的 VFS 调用次数，因此需要使用 -p 标志来指定进程 ID。vfs_* 参数将跟踪该进程的所有 VFS 函数。你会看到 vfs_read () 和 vfs_write () 函数是由 cp 进程调用的。COUNT 列显示该函数被调用的次数：

```C
funccount -p process_ID 'vfs_*'
[root@linuxbox ~]# nohup cp myfile /tmp/myfile &
[1] 1228433
[root@linuxbox ~]# nohup: ignoring input and appending output to 'nohup.out'
[root@linuxbox ~]#
[root@linuxbox ~]# funccount -p 1228433 "vfs_*"
Tracing 66 functions for "b'vfs_*'"... Hit Ctrl-C to end.
^C
FUNC                                    COUNT
b'vfs_read'                             28015
b'vfs_write'                            28510
Detaching...
[root@linuxbox ~]#

```
我们再运行一次，看看在执行简单的复制操作时使用了哪些系统调用。正如预期的那样，在执行 cp 时最常用的系统调用是 read 和 write：

```C
funccount 't:syscalls:sys_enter_*' -p process_ID
[root@linuxbox ~]# nohup cp myfile /tmp/myfile &
[1] 1228433
[root@linuxbox ~]# nohup: ignoring input and appending output to 'nohup.out'
[root@linuxbox ~]#
[root@linuxbox ~]# /usr/share/bcc/tools/funccount -p 1228433 "vfs_*"
Tracing 66 functions for "b'vfs_*'"... Hit Ctrl-C to end.
^C
FUNC                                    COUNT
b'vfs_read'                             28015
b'vfs_write'                            28510
Detaching...
[root@linuxbox ~]#
```

让我们总结一下本节内容。Linux 支持多种文件系统，内核中的 VFS 层确保了这一目标能够轻松实现。VFS 为终端用户进程与不同的文件系统交互提供了标准化的方法。这一标准化是通过实现通用文件模式来实现的。VFS 定义了多个虚拟函数来执行常见的文件操作。由于这种方法，应用程序可以普遍地执行常规文件操作。当一个进程发出系统调用时，VFS 会将这些调用重定向到相应的文件系统函数。


## 解释“一切皆文件”哲学
在 Linux 中，以下所有内容都被视为文件：

- 目录
- 磁盘驱动器及其分区
- 套接字
- 管道
- 光盘

一切皆文件这一说法意味着，Linux 中的所有上述实体都通过文件描述符表示，并通过 VFS 进行了抽象。你也可以说一切都有文件描述符，不过我们不在此展开这个辩论。

特征化 Linux 系统架构的一切皆文件理念也得益于 VFS 的实现。之前我们定义了伪文件系统，它们是动态生成内容的文件系统。这些文件系统也被称为 VFS，并在实现这一理念中发挥了重要作用。

你可以通过procfs伪文件系统获取当前已注册到内核的文件系统列表。在查看这个列表时，注意第一列中有些文件系统会标记为nodev。nodev表示这是一个伪文件系统，不依赖于块设备。像 Ext2、3、4 这样的文件系统是基于块设备创建的，因此它们在第一列中没有nodev标记：
```C
cat /proc/filesystems
[root@linuxbox ~]# cat /proc/filesystems
nodev   sysfs
nodev   tmpfs
nodev   bdev
nodev   proc
nodev   cgroup
nodev   cgroup2
nodev   cpuset
nodev   devtmpfs
nodev   configfs
nodev   debugfs
nodev   tracefs
nodev   securityfs
nodev   sockfs
nodev   bpf
nodev   pipefs
nodev   ramfs
[其余代码省略以简化内容。]
```
你也可以使用mount命令来查看当前系统中已挂载的伪文件系统：
```
mount | grep -v sd | grep -ivE ":/|mapper"
[root@linuxbox ~]# mount | grep -v sd | grep -ivE ":/|mapper"
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,size=1993552k,nr_inodes=498388,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
efivarfs on /sys/firmware/efi/efivars type efivarfs (rw,nosuid,nodev,noexec,relatime)
[其余代码省略以简化内容。]
```
让我们来浏览一下/proc目录。你会看到一长串编号的目录；这些数字代表了系统中当前运行的所有进程的 ID：
```
[root@linuxbox ~]# ls /proc/
1    1116   1228072  1235  1534  196  216  30   54   6  631  668  810      ioports     scsi
10   1121   1228220  1243  1535  197  217  32   55   600  632  670  9      irq         self
1038  1125  1228371  1264  1536  198  218  345  56   602  
633  673  905      kallsyms     slabinfo
1039  1127  1228376  13    1537  199  219  347  570  603  
634  675  91       kcore      softirqs
1040  1197  1228378  14    1538  2    22   348  573  605  635  677  947      keys       stat
1041  12    1228379  1442  16    20   220  37   574  607  
636  679  acpi     key-users  swaps
1042  1205  1228385  1443  1604  200  221  38   576  609  
637  681  buddyinfo  kmsg       sys
1043  1213  1228386  1444  1611  201  222  39   577  610  638  684  bus      kpagecgroup  sysrq-
[其余代码省略以简化内容。]
```
procfs文件系统让我们得以窥见内核的运行状态。/proc中的内容是在我们查看时生成的。这些信息并不是持久保存在你的磁盘上的，所有这一切都发生在内存中。正如你从ls命令中看到的那样，/proc在磁盘上的大小为零字节：
```
[root@linuxbox ~]# ls -ld /proc/
dr-xr-xr-x 292 root root 0 Sep 20 00:41 /proc/
[root@linuxbox ~]#
```
/proc提供了一个实时视图，展示了系统中正在运行的进程。考虑一下/proc/cpuinfo文件。这个文件显示了与你系统相关的处理器信息。如果我们检查这个文件，它会显示为empty：
```
[root@linuxbox ~]# ls -l /proc/cpuinfo
-r--r--r-- 1 root root 0 Nov  5 02:02 /proc/cpuinfo
[root@linuxbox ~]#
[root@linuxbox ~]# file /proc/cpuinfo
/proc/cpuinfo: empty
[root@linuxbox ~]#
```
然而，当通过cat查看文件内容时，会显示大量信息：
```
[root@linuxbox ~]# cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 79
model name      : Intel(R) Xeon(R) CPU E5-2683 v4 @ 2.10GHz
stepping        : 1
microcode       : 0xb00003e
cpu MHz         : 2099.998
cache size      : 40960 KB
physical id     : 0
siblings        : 1
core id         : 0
cpu cores       : 1
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 20
wp              : yes
[其余代码省略以简化内容。]
```
Linux 将所有实体（如进程、目录、网络套接字和存储设备）抽象为 VFS（虚拟文件系统）。通过 VFS，我们可以从内核中获取信息。大多数 Linux 发行版提供多种工具来监控存储、计算和内存资源的消耗。所有这些工具通过procfs中的数据收集各种指标的统计信息。例如，mpstat命令提供关于系统中所有处理器的统计信息，它从/proc/stat文件中获取数据，然后以人类可读的格式呈现这些数据，便于理解：
```
[root@linuxbox ~]# cat /proc/stat
cpu  5441359 345061 4902126 1576734730 46546 1375926 942018 0 0 0
cpu0 1276258 81287 1176897 394542528 13159 255659 280236 0 0 0
cpu1 1455759 126524 1299970 394192241 13392 314865 178446 0 0 0
cpu2 1445048 126489 1319450 394145153 12496 318550 186289 0 0 0
cpu3 1264293 10760 1105807 393854806 7498 486850 297045 0 0 0
[为了简洁，省略了其余的代码。]
```
如果我们在mpstat命令上使用strace工具，它将显示在后台，mpstat使用/proc/stat文件来显示处理器统计信息：
```
strace mpstat 2>&1 |grep "/proc/stat"
[root@linuxbox ~]# strace mpstat 2>&1 |grep "/proc/stat"
openat(AT_FDCWD, "/proc/stat", O_RDONLY) = 3
[root@linuxbox ~]#
```
类似地，常用命令如top、ps和free会从/proc/meminfo文件中收集与内存相关的信息：
```
[root@linuxbox ~]# strace free -h 2>&1 |grep meminfo
openat(AT_FDCWD, "/proc/meminfo", O_RDONLY) = 3
[root@linuxbox ~]#
```
类似于/proc，另一个常用的伪文件系统是/sys。sysfs 文件系统主要包含关于系统硬件设备的信息。例如，要查找系统中磁盘驱动器的信息，如其型号，可以执行以下命令：
```
cat /sys/block/sda/device/model
[root@linuxbox ~]# cat /sys/block/sda/device/model
SAMSUNG MZMTE512
[root@linuxbox ~]#
```
即使是键盘上的 LED 灯也有一个对应的文件在/sys中：
```
[root@linuxbox ~]# ls /sys/class/leds
ath9k-phy0  input4::capslock  input4::numlock  input4::scrolllock
[root@linuxbox ~]#
```
一切皆文件的哲学是 Linux 内核的一个定义性特征。它意味着系统中的一切，包括常规文本文件、目录和设备，都可以通过内核中的 VFS 层进行抽象。因此，所有这些实体都可以通过 VFS 层表示为类似文件的对象。Linux 中有几个伪文件系统，包含有关不同内核子系统的信息。这些伪文件系统的内容仅存在于内存中，并动态生成。

# 总结
Linux 存储栈是一个复杂的设计，由多个层次组成，所有这些层次都协同工作。像其他硬件资源一样，存储位于内核空间。当用户空间程序想要访问这些资源时，它必须调用系统调用。Linux 中的系统调用接口允许用户空间程序访问内核空间中的资源。当用户空间程序想要访问磁盘上的内容时，它首先与 VFS 子系统交互。VFS 提供文件系统相关接口的抽象，并负责在内核中容纳多个文件系统。通过其通用文件系统接口，VFS 拦截来自用户空间程序的通用系统调用（如read()和write()），并将它们重定向到文件系统层中的适当接口。由于这种方式，用户空间程序无需关心底层使用的文件系统，它们可以统一地执行文件系统操作。

本章作为对主要 Linux 内核子系统虚拟文件系统（VFS）的介绍，讲解了其在 Linux 内核中的主要功能。VFS 通过数据结构，如索引节点（inode）、超级块（superblock）和目录项（directory entries），为所有文件系统提供了一个统一的接口。在下一章，我们将深入探讨这些数据结构，并解释它们如何帮助 VFS 管理多个文件系统。