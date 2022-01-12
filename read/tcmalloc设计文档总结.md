# 为什么要求有tcmalloc
对于C++应用程序来说，直接向操作系统发起申请/释放内存的操作开销很大
tcmalloc的本质就是内存的缓存缓存，通过缓存内存降低内存分配的开销
可以做一个简单的类比，redis是缓存，缓存的内容是数据；tcmalloc也是缓存，缓存的是内存


# tcmalloc的基础概念

## Front-end
距离用户代码最近的缓存，分为Per-Thread cache和Per-cpu cache
前端持有自己的内存，访问时不需要加锁
基于线程的缓存主要问题在于，当机器的线程数过多时，占用的内存占用比较大
基于cpu的缓存是比较推荐的使用方式
主要用于分配小内存
大小的设计是比较玄学的，过大的话内存占用过大，过小的话可能需要频繁从middle-end申请内存

## size-class
在实际的业务场景中，应用程序总是会产生各种size的对象
size-class就是把应用申请的对象对齐填充成固定大小
size-class有多个size，从16字节一次递增到262144不等
对内存申请大小的标准化处理，标准化的好处是方便管理
缺点就是会造成一定的空间浪费
举个例子，tcmalloc定义的内存申请和释放的单元是16字节，即使用户申请的内存大小只有12字节，从tcmalloc的视角看，他会把12字节的大小映射成16字节，然后返回内存给用户
这个用户申请size到tcmalloc标准化size的过程是在前端完成的
超大size的对象，需要直接从后端申请

### 为什么要有size-class?
应该主要是面向内存释放与管理场景的设计
假定从一个完整的page持续的分配若干对象，每次从page中获取的对象大小如果都不一致，那么在释放时之后会产生大小不一的内存碎片
如果想对释放后的内存进行复用，寻找合适大小的对象过程会比较麻烦
如果申请的对象内存大小都是统一的，那么可以理解为其实就没有内存碎片，随便选一片释放过的内存区域即可

## 关于tcmalloc中的page
tcmalloc中的page主要是是指一段固定大小的连续内存(4KiB, 8KiB, 32KiB, 256KiB)，不是操作系统的page
page有两种存储对象的方式
1 结合上文的size-class可得，同一个page只能保存同一个尺寸的对象
2 对于比较大的一个page存不下的对象，会由多个page保存，多个page会组成一个span

关于page大小设置的权衡点
page过小可能会导致向OS申请内存的次数过多
page过大则可能会有内存空间的浪费

## Per-Thread Mode
基于线程的缓存，key=线程,value=可分配的内存
数据结构，每个size-class都会对应一个链表
每个线程的缓存大小是有上限的，当某个线程期望扩大自己的最大容量时，他会从其他线程中窃取内存

### PerThread Mode的问题
线程越多，内存开销越大
基于链表的实现寻址效率比价低，cpu cache不太友好

## Per-CPU Mode
基于cpu层次的内存缓存
tcmalloc会划分出一大片内存，然后划分给所有的逻辑cpu
数据结构
1 每个cpu占有这个内存区域的一部分，比如这片内存区域依次划分为cpu region 0/cpu region 1/cpu region 2
2 与size-class的关系，每个cpu region的内部有多个size-class区域，用于分配不同尺寸的对象
2 每个size-class区域内部就是实际用于内存分配的地方，分为header以及objects，header负责记录元数据，objects区域就是实际分配出去的内存；可以简单理解为一个vector<T>，支持写入对象，删除对象，扩容缩容的操作

### Restartable Sequences
主要设计目的是，从per-cpu数组的读写操作都是无锁的，从而提升性能
我觉得是很亮眼的设计，需要看下详细设计

## Middle-end
主要是俩cache，对于这些cache的访问都是加了互斥锁的
1 transfer cache
直接与front-end与central free list交互
允许内存在不同的cpu(thread)之间流动
为啥要有transfer cache

2 central free list per size-class
基于span去管理内存，而一个span是一个或者多个page
直接与back-end交互
当一个span内部的所有对象都回收之后，整个span都会被返回给back-end
对象到span的映射是通过pagemap完成的


## 如何根据指针找到对应的内存区域
当用户申请内存的时候，只需要返回给用户一个指针（tcmalloc里的术语叫虚地址）即可，这个指针是指向一片可用的内存区域的
当用户需要释放内存时，对于tcmalloc来说，需要根据指针找到分配出去的那片内存，把那片内存标记为已释放
那么如何根据指针找到他对应的内存区域在哪呢？

对于大对象（标准是一个page存不下，需要多个page）来说，是用pagemap的保存的，具体的数据结构实现是用radix tree；一组page就是一个span
对于小对象（一个page以内就可以存下的），page内用的是一个类似unrolled linked list保存的


## Back-end
三个职责
1 管理大块未使用的内存
2 当前持有内存无法满足请求时，就从操作系统申请内存
3 返回不需要的内存给操作系统

概括来说就是，持有大块内存，主要负责与操作系统交互

目前的back-end存在历史遗留问题
1 legacy pageHeap，
2 hugepage aware pageheap，以huge page的方式管理内存，主要目的是降低TLB Miss

### Legacy PageHeap
数据结构
1 首先是有一个数组，数组的每个成员都是一个freelist，数组的长度是256
2 freelist的结构，freelist本身是个链表，链表中的每个entry会持有若干个page，第n个freelist，entry中的page数就为n
3 第256个freelist的entry持有的page数是大于256的
4 这里有个细节是，每个entry的page内存应该是连续的

为什么这么设计？
还是出于标准化的设计目的，大内存申请的场景下，一般是申请一个或者多个page的，因此pageheap就提前把按照这个结构存储
申请n个page，就去pageheap数组中找到第n个freelist，然后遍历freelist找到空闲的entry（该entry包含n个page）直接返回

申请n个page，不一定非得在第n个free-list中找，最终选定的free-list位置可以大于n，剩余的那些page可以重新插入到pageheap中

这种设计的弊端


### HugePage Aware PageHeap
这货主要针对hugepage 2M的情况设计的，比如x86系统上一个hugepage为2M
filler cache
region cache


# 相关概念
## TLB是什么？
操作系统需要把进程内的虚拟地址映射成实际的物理地址
此处涉及操作系统页表管理的内容
主要的结论是，如果通过直接访问页表的方式进行地址映射，访问内存的开销是很大（主要相比于直接访问寄存器而言）
因此引入TLB的概念，其实就是页表的缓存，缓存的位置在cpu，访问速度和访问寄存器相当

为什么引入hugepage可以降低cache miss率？
盲猜申请的页面越大，页表数据越小，缓存的利用率也就越高

第二个问题是，直接向操作系统申请内存，是否会被做填充和对齐，最大的page大小是多少？

# 如何工作

# 优缺点

# 一些调优规则

# 对数据库设计的指导意义

# 问题总结
1 为什么要有对齐？不对齐有啥问题吗？


2 为什么要有前中后端的概念？
前端存在的意义，可以看到前端缓存的划分逻辑，要么是线程粒度的，要么是cpu粒度的，可以直接想到的好处是，当申请的内存大小在一定范围内时，可以避免申请内存的竞争开销
如果没有前端，用户直接向中端申请内存，应该会出现比较激烈的竞争
线程和cpu的本质其实就是用户代码实际执行的地方

3 为啥缓存的key是线程/cpu?