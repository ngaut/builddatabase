# TiDB 源码介绍

这篇文章是一篇入门文档，难度系数比较低，其中部分内容可能大家在其他渠道已经看过，不过为了内容完整性，我还是会放在这里。

## TiDB 架构
![](architecture.png "TiDB Architecture")

本次 TiDB 源码之旅从这幅简单的架构图开始，这幅图很多人都看过，我们可以用一句话来描述这个图：『TiDB 是一个支持 MySQL 协议，以某种支持事务的分布式 KV 存储引擎为底层存储的 SQL 引擎』。从这句话可以看出有三个重要的事情，第一是如何支持 MySQL 协议，与 Client 交互，第二是如何与底层的存储引擎打交道，存取数据，第三是如何实现 SQL 的功能。本篇文章会先介绍一些 TiDB 有哪些模块及其功能简要介绍，然后以这三点为线索，将这些模块串联起来。

## 代码简介
TiDB 源码完全托管在 Github 上，从[项目主页](https://github.com/pingcap/tidb "项目主页")可以看到所有信息。整个项目使用 Go 语言开发，按照功能模块分了很多 Package，通过一些依赖分析工具，可以看到项目内部包之间的依赖关系。   
大部分包都以接口的形式对外提供服务，大部分功能也都集中在某个包中，不过有一些包提供了非常基础的功能，会被很多包依赖，这些包需要特别注意。   
项目的 main 文件在 tidb-server/main.go，这里面定义了服务如何启动。整个项目的 Build 方法可以在 [Makefile](https://github.com/pingcap/tidb/blob/source-code/Makefile#L140) 中找到。   
除了代码之外，还有很多测试用例，可以在 xx\_test.go 中找到。另外 `cmd` 目录下面还有几个工具包，用来做性能测试或者是构造测试数据。   

## 模块介绍
TiDB 的模块非常多，这里做一个整体介绍，大家可以看到每个模块大致是做什么用的，想看相关功能的代码是，可以直接找到对应的模块。

| Package | Introduction |
| ------------- | :-------------: |
| ast | 抽象语法树的数据结构定义，例如 `SelectStmt` 定义了一条 Select 语句被解析成什么样的数据结构 |
| cmd/benchdb | 简单的 benchmark 工具，用于性能优化 |
| cmd/benchfilesort | 简单的 benchmark 工具，用于性能优化 |
| cmd/benchkv | Transactional KV API benchmark 工具，也可以看做 KV 接口的使用样例 |
| cmd/benchraw | Raw KV API benchmark 工具，也可以看做不带事务的 KV 接口的使用样例 |
| cmd/importer | 根据表结构以及统计信息伪造数据的工具，用于构造测试数据 |
| config | 配置文件相关逻辑 |
| context | 主要包括 Context 接口，提供一些基本的功能抽象，很多包以及函数都会依赖于这个接口，把这些功能抽象为接口是为了解决包之间的依赖关系 |
| ddl | DDL 的执行逻辑 |
| distsql | 对分布式计算接口的抽象，通过这个包把 Executor 和 TiKV Client 之间的逻辑做隔离 |
| domain | domain 可以认为是一个存储空间的抽象，可以在其中创建数据库、创建表，不同的 domain 之间，可以存在相同名称的数据库，有点像 Name Space。一般来说单个 TiDB 实例只会创建一个 Domain 实例，其中会持有 information schema 信息、统计信息等。 |
| executor | 执行器相关逻辑，可以认为大部分语句的执行逻辑都在这里，比较杂，后面会专门介绍 |
| expression | 表达式相关逻辑，包括各种运算符、内建函数 |
| expression/aggregation | 聚合表达式相关的逻辑，比如 Sum、Count 等函数 |
| infoschema | SQL 元信息管理模块，另外对于 Information Schema 的操作，都会访问这里 |
| kv | KV 引擎接口以及一些公用方法，底层的存储引擎需要实现这个包中定义的接口 |
| meta | 利用 structure 包提供的功能，管理存储引擎中存储的 SQL 元信息，infoschema/DDL 利用这个模块访问或者修改 SQL 元信息 |
| meta/autoid | 用于生成全局唯一自增 ID 的模块，除了用于给每个表的自增 ID 之外，还用于生成 |
| metrics | Metrics 相关信息，所有的模块的 Metrics 信息都在这里 |
| model | SQL 元信息数据结构，包括 DBInfo / TableInfo / ColumnInfo / IndexInfo 等 |
| mysql | MySQL 相关的常量定义 |
| owner | TiDB 集群中的一些任务只能有一个实例执行，比如异步 Schema 变更，这个模块用于多个 tidb-server 之间协调产生一个任务执行者。每种任务都会产生自己的执行者。 |
| parser | 语法解析模块，主要包括词法解析 (lexer.go) 和语法解析 (parser.y)，这个包对外的主要接口是 Parse()，用于将 SQL 文本解析成 AST |
| parser/goyacc | 对 GoYacc 的包装 |
| parser/opcode | 关于操作符的一些常量定义 |
| perfschema | Performance Schema 相关的功能，默认不会启用 |
| plan | 查询优化相关的逻辑 |
| privilege | 用户权限管理接口|
| privilege/privileges | 用户权限管理功能实现 |
| server | MySQL 协议以及 Session 管理相关逻辑 |
| sessionctx/binloginfo | 向 Binlog 模块输出 Binlog 信息 |
| sessionctx/stmtctx | Session 中的语句运行时所需要的信息，比较杂 |
| sessionctx/variable | System Variable 相关代码 |
| statistics | 统计信息模块 |
| store | 储存引擎相关逻辑，这里是存储引擎和 SQL 层之间的交互逻辑 |
| store/mockoracle | 模拟 TSO 组件 |
| store/mockstore | 实例化一个 Mock TiKV 的逻辑，主要方法是 NewMockTikvStore，把这部分逻辑从 mocktikv 中抽出来是避免循环依赖 |
| store/mockstore/mocktikv | 在单机存储引擎上模拟 TiKV 的一些行为，主要作用是本地调试、构造单元测试以及指导 TiKV 开发 Coprocessor 相关逻辑 |
| store/tikv | TiKV 的 Go 语言 Client |
| store/tikv/gcworker | TiKV GC 相关逻辑，tidb-server 会根据配置的策略向 TiKV 发送 GC 命令 |
| store/tikv/oracle | TSO 服务接口 |
| store/tikv/oracle/oracles | TSO 服务的 Client |
| store/tikv/tikvrpc | TiKV API 的一些常量定义 |
| structure | 在 Transactional KV API 上定义的一层结构化 API，提供 List/Queue/HashMap 等结构 |
| table | 对 SQL 的 Table 的抽象|
| table/tables | 对 table 包中定义的接口的实现 |
| tablecodec | SQL 到 Key-Value 的编解码，每种数据类型的具体编解码方案见 `codec` 包 |
| terror | TiDB 的 error 封装 |
| tidb-server | 服务的 main 方法 |
| types | 所有和类型相关的逻辑，包括一些类型的定义、对类型的操作等 |
| types/json | json 类型相关的逻辑 |
| util | 一些实用工具，这个目录下面包很多，这里只会介绍几个重要的包 |
| util/admin | TiDB 的管理语句（ `Admin` 语句）用到的一些方法 |
| util/charset | 字符集相关逻辑 |
| util/chunk | Chunk 是 TiDB 1.1 版本引入的一种数据表示结构。一个 Chunk 中存储了若干行数据，在进行 SQL 计算时，数据是以 Chunk 为单位在各个模块之间流动 |
| util/codec | 各种数据类型的编解码 |
| x-server | X-Protocol 实现|
## 从哪里入手
粗看一下 TiDB 有 80 个包，让人觉得无从下手，不过并不是所有的包都很重要，另外一些功能只会涉及到少量包，从哪里入手去看源码取决于看源码的目的。
如果是想了解某一个具体的功能的实现细节，那么可以参考上面的模块简介，找到对应的模块即可。
如果是相对源码有全面的了解，那么可以从 tidb-server/main.go 入手，看 tidb-server 是如何启动，如何等待并处理用户请求。再跟着代码一直走，看 SQL 的具体执行过程。另外一些重要的模块，需要看一下，知道是如何实现的。辅助性的模块，可以选择性的看一下，有大致的印象即可。

## 重要模块
在全部 80 个模块中，下面几个模块是最重要的，希望大家能仔细阅读，针对这些模块，我们也会用专门的文章来讲解，等所有的文章都 Ready 后，我将下面的表格中的 TODO 换成对应的文章连链接。

| Package |  Related Articles |
| ------------- | ------------- |
| plan |  TODO |
| expression | TODO |
| executor | TODO |
| distsql | TODO |
| store/tikv | TODO |
| ddl | TODO |
| tablecodec | TODO |
| server | TODO |
| types | TODO |
| kv | TODO |
| tidb | TODO |

## 辅助模块
除了重要的模块之外，余下的是辅助模块，但并不是说这些模块不重要，只是锁这些模块并不在  SQL 执行的关键路径上，我们也会用一定的篇幅描述其中的大部分包。

## SQL 层架构
![](tidb-core.png "TiDB SQL Layer")
这幅图比上一幅图详细很多，大体描述了 SQL 核心模块，大家可以从左边开始，顺着箭头的方向看。

### Protocol Layer
最左边是 TiDB 的 Protocol Layer，这里是与 Client 交互的接口，目前 TiDB 只支持 MySQL 协议，相关的代码都在 `server` 包中。   
这一层的主要功能是管理客户端 connection，解析 MySQL 命令并返回执行结果。具体的实现是按照 MySQL 协议实现，具体的协议可以参考 [MySQL 协议文档](https://dev.mysql.com/doc/internals/en/client-server-protocol.html)。这个模块我们认为是当前实现最好的一个 MySQL 协议组件，如果大家的项目中需要用到 MySQL 协议解析、处理的功能，可以参考或引用这个模块。

连接建立的逻辑在 server.go 的 [Run()](https://github.com/pingcap/tidb/blob/source-code/server/server.go#L236) 方法中，主要是下面两行：   
> 236:         conn, err := s.listener.Accept()   
> 258:         go s.onConn(conn)   

单个 Session 处理命令的入口方法是调用 clientConn 类的 [dispatch 方法](https://github.com/pingcap/tidb/blob/source-code/server/conn.go#L465)，这里会解析协议并转给不同的处理函数。

### SQL Layer
大体上讲，一条 SQL 语句需要经过，语法解析--\>合法性验证--\>制定查询计划--\>优化查询计划--\>根据计划生成查询器--\>执行并返回结果 等一系列流程。这个主干对应于 TiDB 的下列包：

| Package |  作用 |
| ------------- | ------------- |
| tidb | Protocol 层和 SQL 层之间的接口|
| parser | 语法解析 |
| plan |  合法性验证 +  制定查询计划 + 优化查询计划 |
| executor | 执行器生成以及执行 |
| distsql | 通过 TiKV Client 向 TiKV 发送以及汇总返回结果 |
| store/tikv | TiKV Client |

### KV API Layer
TiDB 依赖于底层的存储引擎提供数据的存取功能，但是并不是依赖于特定的存储引擎（比如 TiKV），而是对存储引擎提出一些要求，满足这些要求的引擎都能使用（其中 TiKV 是最合适的一款）。   
最基本的要求是『带事务的 Key-Value 引擎，且提供 Go 语言的 Driver』，再高级一点的要求是『支持分布式计算接口』，这样 TiDB 可以把一些计算请求下推到 存储引擎上进行。   
这些要求都可以在 `kv` 这个包的[接口](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go)中找到，存储引擎需要提供实现了这些接口的 Go 语言 Driver，然后 TiDB 利用这些接口操作底层数据。   
对于最基本的要求，可以重点看这几个接口：
* [Transaction](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L121)：事务基本操作
* [Retriever ](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L75)：读取数据的接口
* [Mutator](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L91)：修改数据的接口
* [Storage](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L229)：Driver 提供的基本功能
* [Snapshot](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L214)：在数据 Snapshot 上面的操作
* [Iterator](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L255)：`Seek` 返回的结果，可以用于遍历数据

有了上面这些接口，可以对数据做各种所需要的操作，完成全部 SQL 功能，但是为了更高效的进行运算，我们还定义了一个高级计算接口，可以关注这三个 Interfce/struct :
* [Client](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L150)：向下层发送请求以及获取下层存储引擎的计算能力
* [Request](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L176): 请求的内容
* [Response](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L204): 返回结果的抽象

### 小结
至此，读者已经来了解了 TiDB 的源码结构以及三个主要部分的架构，更详细的内容会在后面的章节中详细描述。

