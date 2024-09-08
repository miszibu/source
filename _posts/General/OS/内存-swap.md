---
title: 内存-swap
date: 2020-04-23 22:26:48
tags: [OS, Memory]
---

内存 **Swap**是指：当物理内存不够时，系统将就会把一些不常用的内存页交换到 swap 空间（磁盘），以释放内存。

Swap 在提高内存使用上限的同时，也会由于磁盘本身的读写速率的问题，一定程度的降低内存读写效率。因此请根据具体情况使用 Swap 内存，且不宜超过物理内存的50%。

<!--more-->

## Swapping 定义

Linux 将物理内存 **RAM**（Random Access Memory）划分为称之为内存页（**Page**）的小块。Swapping 指的就是将一部分内存页从内存拷贝到我们称之为 Swap 空间的磁盘区域，从而释放内存页。这部分物理内存和 Swap 空间我们称之为**虚拟内存**。

> 虚拟内存 = Swap Space + 部分物理内存

## Swapping 作用

Swapping 有以下俩个作用，

1. 当系统实际需要内存大于物理内存上限时，内核将部分低使用率的内存页交换到磁盘上去，从而腾出足够的内存空间给急需内存的应用和程序。
   * 举例说：我们启动在只有5GB的可用内存情况下，启动一个-xms -xmx = 6GB 的 Java 程序，那么不足的1GB，将由系统通过 Swapping 腾出。
2. 在应用启动时，往往需要大量的内存空间来完成初始化，而这一部分空间在之后的使用率却很低，系统则能swap这部分内存页给其他应用，甚至于磁盘缓存。

## Swapping 的不足

> 当 Swpping 频繁发生的时候，增大RAM 是最好的选择。

然而, Swapping 也存在不足，因为磁盘的读写速度与内存相比存在着巨大的差距。内存的读写是纳秒级别的，磁盘的读写是毫秒级别的，这里存在上千倍的差距。访问磁盘的速度和访问内存的速度差距十分明显。

当 Swapping 频繁发生的时候，系统的performance 会明显下降。

有时候，内存页被频繁换入换出，系统在艰难的从内存中寻找空闲空间来保持应用的正常运行。在这个情况下，最好的方式就是增大 RAM。

为什么呢，因为系统会优先选择低使用率的内存页来进行 Swap，然而频繁的 Swap 表示，内存资源紧俏，不存在低使用率的内存页了。

## Swapping 空间类型

Linux存在两种不同类型的 Swap 空间，

* Swap Partition: Swap分区是磁盘中一块只用于swapping 使用的单独分区。
* Swap File: 一个特殊的文件，介于系统文件和数据文件之间

## 查看Swap 空间类型

使用 `swapon -s` 命令来查看系统使用的 swap 空间类型，输出如下

```
Filename  Type       Size       Used Priority
/dev/sda5 partition  859436  0       -1
```

每一行代表着一个单独的 Swap 空间。

* Type: 
  * Partition: swap space
  * File: swap file
* Size: 以 KB 为单位
* Used: 已使用的Swapping space size
* Priority: 当同时 mount 多个 swap spaces ，且它们的优先级相同时（往往在不同的设备上），系统会交替安排其 swapping 活动，从而提高 swapping 的 performance.
  * 其实这很好理解，因为 SwappIng 的问题，存在于磁盘和内存空间读取效率的巨大差距，可以对内存页码进行分配，进而提高 N 倍读取效率。

## 如何添加 SwappIng Space

这部分内容不太会涉及，有需要请参考[all-about-linux-swap-space](https://www.linux.com/news/all-about-linux-swap-space/)



## Swap Space 的大小多少为宜

当物理内存足够使用时，不需要Swap Space，系统也能良好运行。但当物理内存不足，系统崩溃时，我们所能做的最佳方案就是使用 swap space，因为磁盘的价格确实便宜。

在旧版本的 类Unix 操作系统中（比如 Sun OS 和 Ultrix)，往往需要两到三倍物理内存大小的虚拟内存空间。

在现代操作系统中，如 Linux，最佳的方案是根据使用场景的不同来划分虚拟内存的大小。

1. 桌面系统：建议 Swap Space 的大小是系统物理内存的两倍，这可以是我们尽可能多的运行APP（空闲 APP 的内存页将被换出）
2. **Server: 建议为系统物理内存的一半**，这样，我们可以获得 swapping 带来的一定灵活性，记得监控 swap space 的使用情况，必要时升级内存。



## Linux Kernal 2.6 ---swappiness 



Linux 内核在2.6版本添加了一个新的内核参数`swappiness` 来管理swap 的方式，值为0-100。

更高的值，内存页的 swap更频繁，相对的，更低的值，内存中将保持更多应用，哪怕当应用处于空闲状态。

swappiness 的值默认时60，使用 root账号运行以下命令，可以临时改变这个参数（直到下次重启）

```
echo 50 > /proc/sys/vm/swappiness
```

通过修改在/etc/sysctl.conf 中的 vm.swappiness 的值来永久修改该参数

## Reference

[all-about-linux-swap-space/](https://www.linux.com/news/all-about-linux-swap-space/)