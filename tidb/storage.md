# TiDB 存储引擎接入指南

## 简介

TiDB 在架构上被设计成分层的结构，SQL 逻辑层和 KV 存储层被划分得比较清楚。而且 SQL 层只依赖于抽象的接口而不依赖于存储引擎的具体实现，这保证了我们可以比较方便地为 TiDB 接入新的存储引擎，当然存储引擎需要满足一定的条件并正确实现相应的接口。

在 TiDB 的当前版本中，我们实现了 `localstore` 和 `hbase` 两套存储引擎。`kv/kv.go` 文件定义了 Store 所需要实现的抽象接口；`store/localstore` 实现了一份单机存储引擎；`store/hbase` 则依托于 hbase 实现了一份分布式存储引擎。

## 接入步骤

### 1. 实现 Driver

`kv/kv.go` 中定义的 Driver 接口如下：
```go
// Driver is the interface that must be implemented by a KV storage.
type Driver interface {
        // Open returns a new Storage.
        // The path is the string for storage specific format.
        Open(path string) (Storage, error)
}
```
新引擎需要实现一份 Driver 并通过 `tidb.RegisterStore()` 进行注册，注册之后就可以通过 `tidb.NewStore()` 或 `tidb.Open()` 创建存储引擎或数据库连接了。

path 约定使用形如 `engine://path[?param=value&...]` 的 URL。`Driver.Open()` 的实现负责解析 path 并创建对应的存储引擎实例，可参考 `store/hbase/kv.go` 中的实现。

### 2. 实现 Storage

`kv/kv.go` 中定义的 Storage 原型如下：
```go
// Storage defines the interface for storage.
// Isolation should be at least SI(SNAPSHOT ISOLATION)
type Storage interface {
        // Begin transaction
        Begin() (Transaction, error)
        // GetSnapshot gets a snapshot that is able to read any data which data is <= ver.
        // if ver is MaxVersion or > current max committed version, we will use current version for this snapshot.
        GetSnapshot(ver Version) (Snapshot, error)
        // Close store
        Close() error
        // Storage's unique ID
        UUID() string
        // CurrentVersion returns current max committed version.
        CurrentVersion() (Version, error)
}
```

注意 TiDB 要求 Storage 的隔离性应当至少是 [SI（Snapshot Isolation）](https://en.wikipedia.org/wiki/Snapshot_isolation) 级别的，这意味着所有的读写操作的结果与事务开始时刻息息相关。

TiDB 中我们用严格递增的版本 `Version` 来标识不同事务的时序，在 hbase store 的实现里使用了全局时间戳服务来保证这一点，细节可参考 [github.com/ngaut/tso](https://github.com/ngaut/tso)。

具体的，`Begin()` 用于创建一次数据库读写事务（事务使用数据库当前版本标识时序），`GetSnapshot()` 创建指定版本的只读快照，`UUID()` 返回能唯一标识数据库的 string，`CurrentVersion()` 返回数据库当前时刻的版本。

## 3. 实现 Snapshot

Snapshot 用于处理只读事务，`kv/kv.go` 中定义了相关接口如下：

```go
type Key []byte

// Retriever is the interface wraps the basic Get and Seek methods.
type Retriever interface {
        // Get gets the value for key k from kv store.
        // If corresponding kv pair does not exist, it returns nil and ErrNotExist.
        Get(k Key) ([]byte, error)
        // Seek creates an Iterator positioned on the first entry that k <= entry's key.
        // If such entry is not found, it returns an invalid Iterator with no error.
        // The Iterator must be Closed after use.
        Seek(k Key) (Iterator, error)
}

// Iterator is the interface for a iterator on KV store.
type Iterator interface {
        Valid() bool
        Key() Key
        Value() []byte
        Next() error
        Close()
}

// Snapshot defines the interface for the snapshot fetched from KV store.
type Snapshot interface {
        Retriever
        // BatchGet gets a batch of values from snapshot.
        BatchGet(keys []Key) (map[string][]byte, error)
        // Release releases the snapshot to store.
        Release()
}
```

其中 `Get()` 和 `Seek()` 提供了最基本的 KV 读取操作，`Iterator` 用于遍历，`BatchGet()` 用于批量读取。如果对应的引擎没有针对批量读操作的优化，`BatchGet()` 接口可以简单地实现为多次 `Get()` 操作。

## 4. 实现 Transaction

Transaction 用于处理读写事务，`kv/kv.go` 中定义了相关接口如下：

```go
// Mutator is the interface wraps the basic Set and Delete methods.
type Mutator interface {
        // Set sets the value for key k as v into kv store.
        // v must NOT be nil or empty, otherwise it returns ErrCannotSetNilValue.
        Set(k Key, v []byte) error
        // Delete removes the entry for key k from kv store.
        Delete(k Key) error
}

// Transaction defines the interface for operations inside a Transaction.
// This is not thread safe.
type Transaction interface {
        RetrieverMutator
        // Commit commits the transaction operations to KV store.
        Commit() error
        // Rollback undoes the transaction operations to KV store.
        Rollback() error
        // String implements fmt.Stringer interface.
        String() string
        // LockKeys tries to lock the entries with the keys in KV store.
        LockKeys(keys ...Key) error
        // SetOption sets an option with a value, when val is nil, uses the default
        // value of this option.
        SetOption(opt Option, val interface{})
        // DelOption deletes an option.
        DelOption(opt Option)
}
```

其中 `Set()` 和 `Delete()` 定义了基本的 KV 写操作；`Commit()` 和 `Rollback()` 用于事务的提交和回滚；`String()` 返回事务的唯一标识，已有实现中我们直接使用了事务起始版本号；`LockKeys()` 用于加锁指定的 Key，以保证当前事务提交之前不会有其他事务修改相关数据；`SetOption()` 和 `DelOption()` 用于设置一些性能优化选项，存储引擎可根据需要实现。

TiDB 中 `store/localstore` 和 `store/hbase` 的实现都使用了“乐观锁”，即大部分情况下冲突的检测在事务提交阶段进行，如果提交时发生可重试的事务冲突，SQL 层会根据情况选择进行重试。`kv/union_store.go` 中的 `UnionStore` 提供了与之相关的写缓冲、条件检测等相关组件。

## 测试

使用 interpreter 做简单测试。`tidb/interpreter` 目录中有一份简单的交互式 SQL 环境，编译后运行 `./interpreter -store myengine -path mypath`，即可进行简单的 SQL 语句测试，注意需要略微修改代码保证自定义 Driver 被成功注册。

使用 tidb-server 测试。`tidb/tidb-server` 目录中实现了 MYSQL 兼容的服务器，编译后运行 `./tidb-server -store myengine -path mypath -L error`，启动成功后可使用 MYSQL 客户端进行测试，注意需要略微修改代码保证自定义 Driver 被成功注册。

使用 Driver 测试。在新的 Go 工程中使用 `tidb.Open()` 创建数据库连接后进行 SQL 操作。

运行 TiDB 提供的测试用例。`store/store_test.go` 文件中有一些针对 Storage 功能性及一致性的测试，可使用 `go test -teststore myengine -testpath mypath` 进行测试，注意需要略微修改代码保证自定义 Driver 被成功注册。