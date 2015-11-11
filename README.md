# builddatabase
从零开始写分布式数据库，以 [TiDB](https://github.com/pingcap/tidb) 源码为例

*	第一章 概论

*	第二章 基础知识
	*	[数据库的隔离级别]()
	*	[分布式系统的CAP理论]()
	*	[Google Spanner简介]()
	*	[Google F1简介]()
	*	[HBase 简介]()
	*	[Google percolator事务模型精讲]()
	*	[TiDB 的分布式事务模型]()
	*	[TiDB 的源码结构]()
	*	[如何参与 TiDB 开源项目]()
	
*	第三章 SQL解析
	*	[golex]()
	*	[goyacc]()
	*	[解析整个流程]()
	*	[案例：添加一个函数]()
	*	[案例：添加一个语句]()
	
*	第四章 MySQL 协议支持
	*	[协议概述]()
	*	[如何用wireshar来辅助调试]()
	*	[Reques 格式]()
	*	[Response 格式]()
	*	[TiDB 代码分析]()
		 
*	第五章 执行计划优化 	
	* [基于规则的优化]()
	* [基于开销分析的优化]()
	*	[分布式/并行优化]()
	*	[延迟计算]()
	*	[案例分析：FoundationDB, Apache Phoenix]()
	*	[案例分析：GreenPlum]()
	*	[TiDB 优化器代码分析]()
	
*	第五章 分布式 SQL 数据库的异步 Schema 变更 	
	* [为什么在分布式系统中异步变更schema比较困难]()
	* [深度剖析 Google F1的 schema 变更算法]()
	*	[TiDB 的异步 schema 变更实现]()
	
		
* 第五章 高级
	*	[深入分析 Spanner那些相关的论文]()
	*	[一些新的论文和技术讨论]()
