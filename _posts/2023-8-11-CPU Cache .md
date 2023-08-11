> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ylgrgyq.github.io](https://ylgrgyq.github.io/2018/01/15/cache-structure/)

> 最近看了一些 CPU 缓存相关的东西，在这里做一下记录。

最近看了一些 CPU 缓存相关的东西，在这里做一下记录。

[](#一些基本概念 "一些基本概念")一些基本概念
--------------------------

### [](#CPU-缓存出现的原因 "CPU 缓存出现的原因")CPU 缓存出现的原因

1.  主存一般是 DRAM，CPU 速度比主存快很多倍，没有缓存存在时 CPU 性能很大程度取决于读取存储数据的能力
2.  比 DRAM 快的存储介质是存在的，比如作为 CPU 高速缓存的 SRAM，只是很贵，做很大的 SRAM 不经济
3.  CPU 访问数据存在时间局部性和空间局部性，所以可以将 CPU 需要频繁访问的少量热数据放在速度快但很贵的 SRAM 中，既能大幅度改善 CPU 性能也不会让成本提升太多

### [](#Cache-Line "Cache Line")Cache Line

CPU 每次访问数据时先在缓存中查找一次，找不到则去主存找，访问完数据后会将数据存入缓存，以备后用。这就产生了一个问题，CPU 在访问某个地址的时候如何知道目标数据是在缓存中存在？如何知道缓存的数据是否还有效没被修改？不能为每个存入缓存的字节都打标记，所以 CPU 缓存会划分为固定大小的 block 称为 cache line，作为存取数据的最小单位。大小都为 2 的整数幂，比如 16 字节，256 字节等。这样一个 cache line 这一整块内存能通过一个标记来记录是否在内存中，是否还有效，是否被修改等。一次存取一块数据也能充分利用总线带宽以及 CPU 访问的空间局部性。

### [](#Cache-Write-Policy "Cache Write Policy")Cache Write Policy

Cache 不光是在读取数据时有用，目前大部分 CPU 在写入数据时也会先写 Cache。一方面是因为新存数据很可能会被再次使用，新写数据先写 Cache 能提高缓存命中率；另一方面 CPU 写 Cache 速度更快，从而写完之后 CPU 可以去干别的事情，能提高性能。

CPU 写数据如果 Cache 命中了，则为了保持 Cache 和主存一致有两种策略。如果 CPU 写 Cache 每次都要更新主存，则称为 **Write-Through**，因为每次写 Cache 都伴随主存更新所以性能差，实际使用的也少；写 Cache 之后并不立即写主存而是等待一段时间能积累一些改动后再更新主存的策略称为 **Write-Back**，性能更好但为了保证写入的数据不丢使机制更加复杂。采用 Write Back 方式被修改的内存在从 Cache 移出 (比如 Cache 不够需要腾点空间) 时，如果被修改的 cache line 还未写入主存需要在被移出 Cache 时更新主存，为了能分辨出哪些 Cache 是被修改过哪些没有，又需要增加一个新的标志位在 Cache line 中去标识。

CPU 写数据如果 Cache 未命中，则只能直接去更新主存。但更新完主存后又有两个选择，将刚修改的数据存入 Cache 还是不存。每次直接修改完主存都将主存被修改数据所在 Cache Line 存入 Cache 叫做 **Write-Allocate**。需要注意的是 Cache 存取的最小单位是 Cache Line。即使 CPU 只写一个字节，也需要将被修改字节所在附近 Cache Line 大小的一块内存完整的读入 Cache。如果 CPU 写主存的数据超过一个 Cache Line 大小，则不用再读主存原来内容，直接将新修改数据写入 Cache。相当于完全覆盖主存之前的数据。

[](#CPU-缓存结构 "CPU 缓存结构")CPU 缓存结构
--------------------------------

### [](#Direct-Mapped-Cache "Direct Mapped Cache")Direct Mapped Cache

最简单的缓存结构就是 Direct Mapped 结构。如下图所示，每个 cache line 由基本的 valid 标志位，tag 以及 data 组成。当访问一个内存地址时，根据内存地址用 Hash 函数处理得到目标地址所在 cache line 的 index。根据 index 在 Cache 中找到对应 cache line 的 data 数据，再从 data 中根据内存地址的偏移量读取所需数据。因为 Hash 函数是固定的，所以每一个内存地址在缓存上对应固定的一块 cache line。所以是 Direct Mapped。

实际中为了性能 hash 函数都非常简单，就是从内存地址读取固定的几个 bit 数据作为 cache line 的 index。拿下图为例，cache line 大小为 4 字节，一共 32 bit 是图中的 Data 字段。4 字节一共需要 2 bit 用于寻址，所以看到 32 bit 的地址中，0 1 两个 bit 作为 offset。2 ~ 11 bit 作为 cache line 的 index 一共 1024 个，12 ~ 31 bit 作为 tag 用于区分映射到相同 cache line 的不同内存 block。比如现在要读取的地址是 0x1124200F，先从地址中取 2 ~ 11 bit 得到 0x03 表示目标 cache line 的 index 是 3，之后从地址中读 12 ~ 31 bit 作为 tag 是 0x11242。拿这个 tag 跟 index 为 3 的 cache line 的 tag 做比较看是否一致，一致则表示当前 cache line 中包含目标地址，不一致则表示当前 cache line 中没有目标地址。因为 cache 比内存小很多，所以可能出现多个不在同一 cache line 的内存地址被映射到同一个 cache line 的情况，所以需要用 tag 做区分。最后，如果目标地址确实在 cache line，且 cache line 的 valid 为 true，则读取 0x1124200F 地址的 0 ~ 1 bit，得到 0x03 表示读取当前 cache line 中最后一个字节的数据。

[![](https://ylgrgyq.github.io/2018/01/15/cache-structure/direct-mapped.png)](https://ylgrgyq.github.io/2018/01/15/cache-structure/direct-mapped.png "图来自：http://opass.logdown.com/posts/249025-discussion-on-memory-cache")图来自：http://opass.logdown.com/posts/249025-discussion-on-memory-cache

上图的 Cache 是 1024 X 4 字节 一共 4 KB。但由于 Tag 和 Valid 的存在，缓存实际占用的空间还会更大。

#### [](#替换策略 "替换策略")替换策略

因为 Direct Mapped 方式下，每个内存在 Cache 中有固定的映射位置，所以新访问的数据要被存入 Cache 时，根据数据所在内存地址计算出 Index 发现该 Index 下已经存在有效的 Cache Line，需要将这个已存在的有效 Cache Line 从 Cache 中移出。如果采用 Write-Back 策略，移出时需要判断这个数据是否有被修改，被修改了需要更新主存。

### [](#Two-way-Set-Associative-Cache "Two-way Set Associative Cache")Two-way Set Associative Cache

我们希望缓存越大越好，越大的缓存经常意味着更快的执行速度。对于 Direct Mapped Cache 结构，增大缓存就是增加 Index 数量，相当于是对上面表进行纵向扩展。但除了纵向扩展之外，还可以横向扩展来增加 Cache 大小，这就是 Two-way Set Associative Cache。

基本就是如下图所示，图上省略了 Tag 和 Valid 等标识每个 cell 就是一个 cache line，与 Direct Mapped Cache 不同点在于，Two-way Set Associative Cache 里每个 index 对应了两个 cache line，每个 cache line 有自己的 tag。同一个 index 下的两个 cache line 组成 Set。在访问内存时，先根据内存地址找到目标地址所在 Set 的 index，之后并发的去验证 Set 下的两个 cache line 的 tag，验证目标地址是否在 cache line 内，在的话就读取数据，不在则去读主存。

这里并发的验证两个 cache line 的 tag 是由硬件来保证，硬件电路结构会更加复杂但查询效率与 Direct Mapped Cache 一致。

[![[assets/img/d90fbe39eade6c855d4b35c065549564_MD5.png]]](https://ylgrgyq.github.io/2018/01/15/cache-structure/two-way.png "图来自：PerfBook https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html")图来自：PerfBook https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html

Set 内的两个 cache line 是具有相同 index 的两个不同 cache line。比如还是按 0x1124200F 目标地址来举例，该地址和 0x9124200F 地址的 2 ~ 11 bit 位完全相同，即从 0x1124200C 到 0x1124200F 这 4 个字节的内存，和从 0x9124200C 到 0x9124200F 这 4 个字节的内存映射到了 cache 中的同一个 index 下。这两个 cache line 能放在一个 Set 内，在访问时能同时被比较 tag。

#### [](#替换策略-1 "替换策略")替换策略

新数据要存入 Cache 时，根据数据所在内存地址计算出 Index 后发现该 Index 下两个 Way 的 Cache Line 都已被占用且处在有效状态。需要有办法从这两个 Cache Line 里选一个出来移除。Direct Mapped 因为一个 Index 下只有一个 Cache Line 就没这个问题。

如果是这里说的 Two-way Set Associative Cache 还比较好弄，给每个 Way 增加一个最近访问过的标识。每次一个 Way 被访问就将最近访问位置位，并清理另一个 Way 的最近访问位。从而在执行替换时，将不是最近访问过的那个 Way 移除。不过下面会看到 N-way Set Associative Cache 当有 N 个 Way 的时候替换策略更加复杂，一般是尽可能使用最少的状态信息实现近似的 LRU。

### [](#N-way-Set-Associative-Cache "N-way Set Associative Cache")N-way Set Associative Cache

顾名思义，就是在 Two-way Set Associative Cache 基础上继续横向扩展，在一个 Set 内加入更多更多的 Way 也即 cache line。这些 cache line 能被并发的同时验证 Tag。如果 Cache 内所有的 cache line 都在同一个 Set 内，即所有 cache line 都能同时被验证 Tag，则这种 Cache 叫做 **Fully Associative Cache**。可以看出 Fully Associative Cache 性能是最强的，能省去从地址中读取 index 查找 Set 的过程。但横向扩展的 Way 越多，结构越复杂，成本越高，越难实现大的 Cache。所以 Fully Associative Cache 虽然存在，但都很小，一般用在 TLB 上。

### [](#Cache-结构为什么发展出横向扩展？ "Cache 结构为什么发展出横向扩展？")Cache 结构为什么发展出横向扩展？

这个是我自己提出来的问题。对 Direct Mapped Cache 扩展 Cache 时就是增加更多的 index，让 cache 表变得更长。但为什么会发展出 Two-way Set Associative Cache 呢？比如如果一共 16 个 cache line，是 16 行 cache line 还是 8 行 Set 每个 Set 两个 cache line 在容量和命中率上似乎并没有差别。

后来看到了[这个问题](https://stackoverflow.com/questions/33314115/whats-the-difference-between-conflict-miss-and-capacity-miss)，明白了其中的原因。主要是需要区分出来 conflict miss 和 capacity miss。当 cache 容量足够，但由于两块不同的内存映射到了同一个 cache line，导致必须将老的内存块剔除产生的 miss 叫做 conflict miss，即使整个 Cache 都是空的，只有这一个 cache line 有效时也会出现 miss。而由于容量不足导致的 miss 就是 capacity miss。比如 cache 只有 32k，访问的数据有 200k，那访问时候一定会出现后访问的数据不断的把先访问数据从 cache 中顶出去，导致 cache 一直处在 miss 状态的问题。

在 capacity miss 方面横向扩展和纵向扩展没有什么区别，主要区别就是 conflict miss。假若轮番访问 A B 两个内存，这两个内存映射到同一个 cache line 上，那对于 Direct Mapped Cache，因为每块内存只有固定的一个 cache line 能存放，则会出现持续的 conflict miss，称为 cache thrashing。而 Two-way Set Associative Cache 就能将 A B 两块内存放入同一个 Set 下，就都能 Cache 住，不会出现 conflict miss。这就是横向扩展的好处，也是为什么横向扩展即使困难，各个 CPU 都在向这个方向发展。并且横向扩展和纵向扩展并不冲突，Two-way Set Associative Cache 也能加多 Set 来进行扩展。

[](#为什么缓存存取速度比主存快 "为什么缓存存取速度比主存快")为什么缓存存取速度比主存快
-----------------------------------------------

[Why is SRAM better than DRAM? - Quora](https://www.quora.com/Why-is-SRAM-better-than-DRAM)

[](#参考 "参考")参考
--------------

[PerfBook](https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html)  
[淺談 memory cache « Opass’s Blog](http://opass.logdown.com/posts/249025-discussion-on-memory-cache)  
[Computer Architecture - Class notes](http://www.cs.iit.edu/~virgil/cs470/Book/)  
《现代体系结构上的 UNIX 系统》