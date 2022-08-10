# 问题

	1 如果parquet + 辅助结构的查询性能可以和olap引擎可比较的话，那么olap引擎存在的意义是啥呢
	2 directly-accessible open format到底是啥？
		可以被客户端直接访问的存储格式
	3 cloud-native DBMS与serverless引擎是啥意思

# 愿景
替代现有的数仓架构
	
	1 开放的数据格式（为了2）
	2 支持机器学习和数据科学
	3 极致的性能

感觉以上其实也是lakehouse的定义

# 现有问题分析

schema-on-write
	
	写入时就做schema的校验

schema-on-read
	
	读取时校验schema，把数据质量校验的问题抛给下游


现有的两层架构
	
	data lake（S3，ADLS，GCS） + 下游数仓（snowflake，redshift）
	data lake的问题在于不支持acid，性能也不行
	数仓的问题在于没法支持机器学习什么的

问题

	Reliability，data lake和数仓的数据比较难保持一致
	Data staleness，数仓中的数据相比于data house中的存在延迟
	Limited support for advanced analytics，高级分析指的是机器学习分析，目前这类分析无法很好的跑在数仓之上
	Total cost of ownership，数据需要存多份，data lake一份，数仓一份，并且数仓的数据也无法直接被机器学习使用

Lake house的时代已经到来，原因是以下问题被解决
	
	1 使用delta lake和apache iceberg提供事务语义
	2 支持机器学习和数据科学直接读取lakehouse中的数据
	3 提供最先进的sql性能，主要通过对常用的parquet + orc的格式加一些辅助数据结构来实现

# 动机：数仓的挑战


# 拓展思考：OLAP引擎和lakehouse的区别是啥