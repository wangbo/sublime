# 期望能够解答的一些问题

1 单机的资源管理实现
1.2 是否有做进程内的隔离
	有做进程隔离

1.3 如果进程内做了资源隔离的话，是硬隔离还是软隔离
	支持基于cgroup的硬隔离，这块代码是有的，具体参考InternalResourceGroup
	但是实际上看代码并没有生效，主要是体现在两个地方
		其一是，初始化资源管理器的代码被注释掉了，参考Context::initResourceGroupManager
			该方法内部不仅会初始化基于cgroup的资源组，也会初始化vw资源组
			说明目前资源组功能可能不太稳定
		其二是cgroup独有的池根本就没有初始化，参考InternalResourceGroup::initCpu
			该方法是生成一个pool，然后把crgoup传给这个pool，pool中的线程都会受到cgroup的约数
			但是目前全局搜索代码，initCpu这个方法并没有被调用

	也就是说，这个功能目前有点像摆设，并没有实际被用起来，比较ppt

1.4 同一个资源组内是否做了调度策略
	没有做自定义的调度策略，而是提交到一个自定义线程池，该线程池的实现和boost::threadpool类似

目前感觉上byconity的单机资源隔离不是个很成熟的功能

1.5 cpu和内存运行时的用量是如何收集的

1.6 查询运行前的校验是如何做的

1.7 执行器运行时的并发是如何确定的

1.8 为什么ResourceGroup的定义是拆分成了VW和Internal的呢？

1.9 执行器，DAG？

1.10 如何避免同一个节点被调度过多查询？

2 目前分布式的资源管理是如何实现的
	


# 现有逻辑梳理

## 基本概念

VirtualWarehouse
	应该就是照抄的snowflake的概念，类似一个独立的集群

ResourceGroup，虚拟资源组概念
	InternalResourceGroup，支持cgroup
	VWResourceGroup，不支持cgroup
	why？

## ResourceGroup初始化的过程

## 查询流程梳理

1 server启动，参考server.cpp，根据节点类型初始化不同的模块
1.1 启动服务端接收查询的常驻线程，TCPHandler
1.2 初始化资源管理器

2 根据查询类型生成简单查询解释器(InterpreterSelectQuery)和复杂查询解释器(InterpreterDistributedStages)
	简单查询指的是scatter/gather就可以完成的
	复杂查询应该是带shuffle的

3 分配虚拟仓库(VirtualWarehouse)，代码入口 trySetVirtualWarehouseAndWorkerGroup
	看起来是根据sql中表所属的db确定vw，然后把vw设置到当前上下文中，参考query_scheduler.pickWorkerGroup
	这里一个vw会对应多个workergroup，所以确定了vw之后会有个选择workergroup的过程
	1 过滤出满足资源需求的workergroup，一个workergroup会包含多个节点
	2 然后根据一些策略对第一步选出的workergroup进行筛选，比如随机，轮询，低cpu优先，低内存优先

4 选择resourcegroup，参考ProcessList::insert
	根据user，queryid和querytype匹配，选择ResourceGroup
	（todo 这些匹配项的含义目前不清楚，需要再看下）

5 选定特定的group之后，做资源的校验，看剩余资源是否足够，参考IResourceGroup::run
	对于internalGroup，主要看内存，并发运行的查询个数（？这块也没理解，并发查询个数是啥意思，这个和队列个数是不是冲突了）以及队列中的查询是否达到最大
	对于VW则更加复杂，内存和并发查询数，但是计算逻辑引入了当前资源组的worker节点数(why？todo，梳理下代码细节）

	这里有几个细节需要注意
	1 只检查了内存，但是没有校验cpu
	2 内存的检查只检查了当前使用中的内存，即将执行的查询可能用到的内存没有参与检查
		但是在通过校验后，会加一个最小的内存用量给当前的group
	3 group的逻辑是有层级结构的，每个resourcegroup都会有一个parent
	4 对于InternalGroup，可以理解为有进程内的资源隔离，但是VWResourceGroup，可以理解为是没有做进程内的强隔离的，支持做了用量检查

	如果资源充足可以运行，那么该查询可以被执行
	但是如果资源不充足会进入等待的逻辑，等待会有超时时间的限制

6 查询运行时，参考PipelineExecutor::executeImpl
	如果是InternalResourceGroup，那么就把pipeline提交到当前group的线程池中，这个线程池是group粒度的，并且支持cgroup
	如果是VWResourceGroup，那么就把pipeline提交到一个全局的线程池中（一个简单的类似boost::threadpool）

结论：
	单机的资源隔离从设计上来说，唯一可取的点是不同group之间可以利用cgroup做隔离
	但是单group内部的表现应该会比较差劲，并发性能可能还不真如doris，因为所有查询共享一个线程池
	调度策略相当于是委托了第三方库

	从实现上看，基于Cgroup的resourceGroup目前应该没生效，看起来不是一个特别靠谱的功能

	对于VWResourceGroup，相当于只在单机做了查询并发数的校验和内存使用的校验，比较简单
	byconty单机的资源隔离整体实现目前看起来还是比较粗糙的

	另外还有个问题是，snowflake其实是基于进程的查询模型，每次查询建一个新的进程，byconity不知道为啥没有照抄




	