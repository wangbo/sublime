# Druid: A Real-time Analytical Data Store

## Abstract
* 特点
	* 列存
	* shared nothing（其实按照之前黄东旭的说的标准，druid应该主要是存储和计算分离的，然后本地会有一些缓存）
	* 高级索引结构
	* 亚秒

## 设计目标
* 高可用
* 查询高并发
* 查询低延迟
* 实时的数据摄入

## Architecture
* 设计目标
	* 低复杂度
	* 组件之间相互独立，保证最小的交互

### 实时节点
* 摄入内存
	* 实时写入的数据会保存在内存中，行存（JVM的堆内内存）
	* 内存中的数据会构建索引
	* 内存中的数据和索引是可查的
	* 问题1：内存中的数据结构是怎样的，是否区分索引和数据，还是说索引和数据都是同一个数据结构
* 持久化磁盘
	* 为了避免内存用满，内存中的数据会被定期的持久化到磁盘上
	* 内存中的数据落盘过程中，会经历行存数据转列存的过程
	* 落盘后的索引会被加载到堆外内存中，因此依然是可查的
* merge(构建segment)
	* 实时节点定期启动后台任务，merge磁盘上的持久化的数据，生成segment
	* merge的时机是当前时间窗口结束的时候
* handoff
	* 把segment上传到```deep storage```
* 时间窗口
	* 实时节点存在一个可配置的等待时间窗口，在当前时间窗口即将结束的时候，会等待一段时间，主要用于等待延迟的数据，目的是为了提高数据的精度
	* 这个设计从本质上区别于doris，非精确的数据，但是实时性较好。doris是绝对精确的数据，不丢弃任何数据，但是实时性较差
* 高可用和扩展性
	* 核心思路是通过外部组件（message bus，一般都是kafka）来实现
	* 故障恢复时间
		* kafka记录了所有的数据，当实时节点挂掉之后，可以从上次提交的偏移量处继续消费，这可以大大降低故障恢复的时间
		* 偏移量提交的时机是当数据持久化到磁盘之后
		* 从这个场景来看，其实kafka承担的主要是一个实时场景数据存储的标准的东西，更像一个适配接口。因为其实还有另一种设计思路是上游客户端直接写入olap引擎
		* 当前druid的这种设计是适配kafka的
		* 问题2：这里比较值得讨论的问题是，kafka这种比较标准化的中间组件，单纯从设计的角度来讲，是否有存在的意义？
	* 扩展性
		* druid实时节点的两种配置方式
			* replication，多个实时节点消费相同的数据，可以避免数据丢失
			* partition，一份上游数据可以分为多个分区，被多个实时节点所消费，可以提升吞吐。

### historical节点
* 主要负责load，serve由实时节点构建的segment
* 架构设计
	* shared noting + 本地缓存
	* 通过zk完成协调工作，比如对segment的load和serve的声明
	* 本地缓存设计
		* 可以加速restart
	* historical节点只处理不可变的数据
* segment只有被加载到内存中之后才可以被查询
	* 应该是是druid查询延迟较低且问题的设计之一
* historical节点实际上只处理不可变的数据结构（感觉上似乎对实时场景有帮助）
	* 最直接的好处
		* 基本上就是读写分离了，数据落到存储层就已经是最终的形态了，后续也不会有compaction啥的
		* 代码实现上也比较简单，相当于doris把compaction去掉，基本就是druid了（？这里存疑，druid是否存在compaction呢）
	* 局限性
		* 功能上局限性较大，没法支持更新
		* 不做compaction的话，查询性能会不会有问题，比如多个segment之间的数据是有交集的
* tier设计
	* 这是一个比较有趣的设计，主要是对historical节点划分分组
	* 比如可以划分配置高的配置低，配置高的存储热数据，配置低的可以存储冷数据
* 问题：historical节点主要是单纯的数据存储，还是会负责一定的计算呢?

### broker节点
* 通过zk感知segment的位置以及是否可以查询，分发和汇总```实时节点```和```history节点```的返回结果，具备一定的聚合能力
*  缓存
	* 缓存的介质，本地堆内存或者外部的key value存储
	* 缓存的内容，segments。（？这里不太确定的是缓存的segment是否需要二次计算，具体业务逻辑是怎样的）
	* 只有history节点的数据需要被缓存，要查询实时数据都是直接路由到实时节点
	* 看起来druid查询的数据基本上都是留在内存中的
* zk挂掉不影响broker节点的运行

### Coordinator节点
* 主要负责管理history节点的数据，包括
	* 加载新的数据
	* 删除过期的数据
	* 多副本复制
	* 保持数据分布的均衡
* 采用多版本并发控制的访问管理segment视图
* coordinator节点需要与Mysql交互，主要保存了以下信息，
	* 所有history节点要服务的segment
	* rule
* rules
	* 主要作用于history节点的segment
	* 作用如下
		* historical节点的segment如何被load和删除
		* segment如何分配到不同的tier
		* 每个tier保持几个副本
* 负载均衡
	* 负载均衡的策略需要了解查询的模式
	* 例如
		* 查询最近的数据，时间范围通常比较连续
		* 而查询较小的segment，速度会比较快
		* 因此时间范围较近的segment，负载均衡会更加频繁。直观上说，应该是最近的数据会被打的比较散吧
	* 基于代价的balance，考虑recency（数据的新鲜度），size；
		* （这个设计其实挺有意思的，doris目前是单纯根据tablet的个数做balance，数据大小算是间接考虑的，至于数据新鲜度，基本不考虑）
* Replication
	* 这个我觉得真的没啥好说的，其实就是coordinator节点负责维护多个副本
* 当zk和mysql挂了之后，coordinator依然可以继续提供有损的服务

# 问题

## 为什么druid的实时性，稳定性以及可用性很好？是否有所牺牲

## druid的segment具体数据结构是怎样的？
是面向内存设计的数据结构还是面向磁盘设计的数据结构
不可变如何理解，有哪些好处与局限性
是行存还是列存

## zookeeper中到底保存了什么信息？

## 对比doris
时间窗口的设计
segment只有加载到内存中之后才是可查的
不可变的数据结构
compaction
为什么目前druid在实时场景要强于doris呢
读写分离

## 值得参考的文献
[31][20]
