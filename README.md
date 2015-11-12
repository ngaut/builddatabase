# 从零开始写分布式数据库
以 [TiDB](https://github.com/pingcap/tidb) 源码为例

*	第一章 概论

*	第二章 基础知识
	*	[数据库的隔离级别]
	*	[select for update or not]
	*	[分布式系统的 CAP 理论]
	*	[Google Spanner 简介]
	*	[Google F1 简介]
	*	[HBase 简介]
	*	[Google percolator 事务模型]
	*	[Yahoo 的 omid 事务模型](omid/README.md)
	*	[TiDB 的分布式事务模型]
	*	[TiDB 的源码结构]
	*	[如何参与 TiDB 开源项目]
	
*	第三章 SQL解析
	*	[词法分析与 golex 用法]
	*	[语法分析与 goyacc 用法]
	*	[解析整个语句的执行流程]
	*	[案例：为 TiDB 添加一个新的函数]
	*	[案例：为 TiDB 添加一个语句]
	*	[思考：如何支持 json/protocol buffer]
	
*	第四章 MySQL 协议支持
	*	[协议概述]
	*	[如何用 wireshark 来辅助调试]
	*	[Request 格式]
	*	[Response 格式]
	*	[Prepare 语句支持]
	*	[TiDB 代码分析]
		 
*	第五章 执行计划优化 	
	* 	[基于规则的优化]
	* 	[基于开销分析的优化]
	*	[分布式/并行优化]
	*	[延迟计算]
	*	[案例分析：FoundationDB, Apache Phoenix]
	*	[案例分析：Google F1, GreenPlum]
	*	[TiDB 优化器代码分析]
	
*	第六章 分布式 SQL 数据库的异步 Schema 变更 	
	*	[为什么在分布式系统中异步变更 schema 比较困难]
	* 	[深度剖析 Google F1 的 schema 变更算法]
	*	[TiDB 的异步 schema 变更实现]
		
*	第七章 高级
	*	[深入分析 Spanner 那些相关的论文]
	*	[一些新的论文和技术讨论]
