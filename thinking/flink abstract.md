# Design idea
## 设计哲学
Flink的设计着认为，```实时的数据分析```,```连续的数据管道```,```批处理```,```迭代算法```都可以通过```pipelined fault-tolerant dataflows```表达和执行

## 业界现状
主要是lambda架构主导，分为两条生产路径:
* 批处理获得高延迟但是精确的结果
* 实时处理获得低延迟但是非精确的结果

Fink认为，```Batch program are special cases of streaming programs```
基于以上问题，flink提出使用同一套框架完成批处理和实时处理，简单来说就是流批统一


# System architecture

## 概念
* runtime
	* distributed dataflow engine which executes dataflow programs
* runtime program
	* a DAG of stateful operators connected with data streams
* core APIs
	* DataSet API，批处理场景
	* DataStream API，实时处理场景

* 处理模型
	* Client作业到JobManager
	* JobManger负责跟踪任务的实际执行情况
	* TaskManager负责实际的执行

和MR的架构很像，没啥好说的

# 重要特性
## Dataflow Graphs
* 所有的flink程序最终都会被编译成dataflow graphs
* 主要包含两部分
	* 有状态的操作
	* data streams，也就是生产者operator和消费者operator流动的数据
* operator本身也是可以有多个并行执行的实例的

## Operator之间的数据交换
* intermediate data stream
	* 是数据交换的核心抽象
	* 代表了生产者产生的数据，并且这个数据可以被消费者消费

* Pipelined intermediate streams
	* 在并发的生产者和消费者之间交换数据
	* 有了这个中间流，消费者可以把back pressure传播到生产者，补偿短期的吞吐量波动（这里其实没太看明白，或者说业务场景不是很理解）
	* 这个中间流主要是用于实时场景

* 补充问题，```back pressure```是啥，什么场景下会有，有啥影响吗？

* Blocking streams
	* 主要用于处理有边界的数据，也就是离线场景
	* 和实时场景不同的是，会缓存生产者所有的数据直到在对消费者可见之前，因此会被划分为不同的stage（这也是和实时场景最大的区别）
	* 感觉上sql引擎不同stage之间的处理逻辑也是同样的道理，这就很抽象了

* 平衡延迟和吞吐
	* 生产者会先把数据写到buffer里，然后buffer中的数据可以发送给消费者
	* 当数据位于buffer中时，有两种处理策略
		* 只要buffer满了，就发送数据给消费者
		* timeout condition reached
	* 通过调大这个buffer可以获得比较高的吞吐
	* 调小timeout可以获得低延迟

* Control Event
	* operator为stream注入各种控制事件，其他operator可以接收到事件并作出响应
	* 主要有以下几类
		* checkpoint barrier
		* watermarks
		* iteration barrier

## 容错
* recovery
	* 在事件流中注入```checkpoint barriers```, 如果上游是kafka的话，那么cb就是kafka中的数据偏移量
	* 如果是中间节点接收到了cb，那么就直接发送给下游
		* 如果中间节点有多个上游，那么会做一个对齐操作，等待所有输入的cb到达之后才会发送给下游数据
	* 当sink节点接收到所有输入流的cb之后，会对当前的快照ack，确认当前偏移量的数据都已经消费完成

## 基于Dataflow的流式分析
* 时间的概念
* watermark
	* 简单说何时不再等待更早的数据
	* 定义了事件时间的处理机制，watermark（t）代表了t之前的数据都已经到达
	* 而当Operator认为t之前的数据都已到达，那么t之前的数据都会别丢弃掉
	* 水位的更新其实也就是时间窗口的更新，所以主要作用还是用于维护一个时间窗口
	* 从设计上来说，水位的设计是延迟和精度的权衡
		* 追求低延迟，那么数据都放内存中-》而内存中只能保留一段时间的数据-》这会损失精度
		* 追求高精度，类似doris2doris的做法，中间节点的数都物化落盘，精度损失较小，但是延迟也更高
		* 只能说解决的业务场景不同，所以设计也不同
	* 产生与消费
		* watermark通常由sourceTask产生，应该是类似直接读数据源的算子
		* 当watermark流过某个Operator时，这个operator的event time会被更新
		* 当一个operator的事件时间被更新之后，会为下游算子构建一个新的水位？（这里为啥不是使用相同的水位直接发送给下游呢）
		* 当一个operator有多个输入流时，它的事件时间和输入流中事件时间最小的那个保持一致
* Stateful stream processing
	* 对于flink来说，状态可以是各种聚合算子，也可以是机器学习中的稀疏矩阵
* Stream Windows
	*  ```Stream windows are stateful operators that assign records to continuously updated buckets kept in memory as part of the operator state```
	* 主要定义了基于窗口的流式计算的三个函数
		* window assigner，主要负责给窗口发数据
		* trigger，可选的；
		* evictor，用于确定哪些数据可以保留在窗口中
	* 为什么flink认为实时场景需要窗口，因为flink认为数据是infinite (unbounded)
	* 但感觉上根本原因是，fink本身也就是个单纯的基于内存的实时计算框架，不物化也就意味着没法支持任意时间段的查询
	* 窗口的滑动方式
		* 基于时间，每30s
		* 基于数据，每处理300个元素
 * watermark和window的关系（待调研）
 	* watermark定义了stream中的处理进度
 	* 而window则是根据watermark进行移动
 	* 基于数据进行窗口的滑动，还需要watermark吗?
* 异步stream iteration
	* 这个主要是用于机器学习场景的，可以暂时不考虑

# 基于Dataflow的批量分析
* 查询优化
	* UDF隐藏了语义，所以传统的优化器没有办法参与优化
	* interesting properties propagation [26]
* 内存管理
	* 对数据进行序列化
	* 直接基于序列化之后的数据进行排序和join
* 批量iteration

# 总结

## 对比doris
从架构实现的角度来说，不具备可比性。
但是可以从要解决的实时场景的问题来考虑，是否具备可替代性和可借鉴

## 与Doris

## 扩展阅读
* 反压
* Chandy-Lamport algorithm for asynchronous distributed snapshots

# 参考
[watermark](http://wuchong.me/blog/2018/11/18/flink-tips-watermarks-in-apache-flink-made-easy/)

[Timely Stream Processing](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/timely-stream-processing.html)

[Stateful Stream Processing](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/stateful-stream-processing.html)

[The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale,Unbounded, Out-of-Order Data Processing](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/43864.pdf)
[Lateness]()

[Distributed Snapshots: Determining Global States of a Distributed System](https://www.microsoft.com/en-us/research/publication/distributed-snapshots-determining-global-states-distributed-system/?from=http%3A%2F%2Fresearch.microsoft.com%2Fen-us%2Fum%2Fpeople%2Flamport%2Fpubs%2Fchandy.pdf)