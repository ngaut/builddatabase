# 从零开始写分布式数据库
以 [TiDB](https://github.com/pingcap/tidb) 和 [TiKV](https://github.com/pingcap/tikv) 源码为例

*	第一章 概论

*	第二章 基础知识
	*	[数据库的隔离级别]
	*	[select for update or not]
	*	[分布式系统的 CAP 理论]
	*	[Google Spanner 简介](spanner/README.md)
	*	[Google F1 简介](f1/README.md)
	*	[HBase 简介]
	*	[Google percolator 事务模型](percolator/README.md)
	*	[Yahoo 的 omid 事务模型](omid/README.md)
	*	[TiDB 的分布式事务模型]
	*	[TiDB 的源码介绍](tidb/sourcecode.md)
	*	[如何参与 TiDB 开源项目](howto/README.md)
	*	[如何添加新的 key value 存储引擎](tidb/storage.md)
	
*	第三章 SQL解析
	*	[词法分析与 golex 用法]
	*	[语法分析与 goyacc 用法]
		*	[通过案例看解决Reduce/Reduce冲突](https://github.com/pingcap/tidb/pull/589/files)
		*	[通过案例看如何解决Shift/Reduce冲突](https://github.com/pingcap/tidb/pull/128/files)
	*	[解析整个语句的执行流程]
	*	[案例：为 TiDB 添加一个新的函数](tidb/builtin.md)
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
	* 	[深度剖析 Google F1 的 schema 变更算法](f1/schema-change.md)
	*	[TiDB 的异步 schema 变更实现](f1/schema-change-implement.md)
		
*	第七章 高级
	*	[Hybird logical clocks](hlc/README.md)
	*	[深入分析 Spanner 那些相关的论文]
	*	[一些新的论文和技术讨论]

*	第八章 实现一个分布式 key-value 引擎
	*	[raft 协议介绍]
	*	[TiKV 的 raft 实现]
	*	[聊聊 Google Spanner]
	*	[TiKV 系统架构]
	*	[TiKV 实现实现跨数据中心容灾]
	*	[TiKV 如何实现自动扩容]
	*	[TiKV 分布式事务实现]
	
*	第九章 如何测试分布式系统
	*	[大话测试]
	*	[用 Jepsen 模拟网络分区,延迟]
	*	[用 Namazu 实现 fault injection]
	*	Fuzz test
