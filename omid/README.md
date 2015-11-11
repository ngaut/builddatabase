# Yahoo 的 omid 事务模型
 
系统架构 [TiDB](https://github.com/yahoo/omid/wiki/images/architecture.png) 

基本概念和名词解释

Timestamp Oracle (TO)

The Server Oracle (TSO)

Commit Table (CT)

Shadow Cells (SCs)

Transactional Clients (TCs) 

操作流程 [TiDB](http://36.media.tumblr.com/3be4620a079c9733bba39d5d23774398/tumblr_inline_nxf4c9gjly1t17fny_500.png) 

冲突检测：

优化细节：
通过记录 key 的 hash 而不是 key 自身来减少存储空间
具体的 hash 算法是murmur3，见代码实现：

public long getCellId() {
        return Hashing.murmur3_128().newHasher()
                .putBytes(table.getTableName())
                .putBytes(row)
                .putBytes(family)
                .putBytes(qualifier)
                .hash().asLong();
    }
