# Yahoo 的 omid 事务模型
 
[系统架构](https://github.com/yahoo/omid/wiki/images/architecture.png) 

基本概念和名词解释

Timestamp Oracle (TO)

The Server Oracle (TSO)

Commit Table (CT)

Shadow Cells (SCs)

Transactional Clients (TCs) 

[操作流程](http://36.media.tumblr.com/3be4620a079c9733bba39d5d23774398/tumblr_inline_nxf4c9gjly1t17fny_500.png) 

冲突检测：

优化细节：
通过记录 key 的 hash 而不是 key 自身来减少存储空间
具体的 hash 算法是murmur3，见代码实现：

```java
   public long getCellId() {
           return Hashing.murmur3_128().newHasher()
                   .putBytes(table.getTableName())
                   .putBytes(row)
                   .putBytes(family)
                   .putBytes(qualifier)
                   .hash().asLong();
       }
```


具体提交流程：

```java
public long handleCommit(long startTimestamp, Iterable<Long> writeSet, boolean isRetry, Channel c) {
        boolean committed = false;
        long commitTimestamp = 0L;

        int numCellsInWriteset = 0;
        // 0. check if it should abort
        if (startTimestamp <= lowWatermark) {
            committed = false;
        } else {
            // 1. check the write-write conflicts
            committed = true;
            for (long cellId : writeSet) {
                long value = hashmap.getLatestWriteForCell(cellId);
                if (value != 0 && value >= startTimestamp) {
                    committed = false;
                    break;
                }
                numCellsInWriteset++;
            }
        }

        if (committed) {
            // 2. commit
            try {
                commitTimestamp = timestampOracle.next();

                if (numCellsInWriteset > 0) {
                    long newLowWatermark = lowWatermark;

                    for (long r : writeSet) {
                        long removed = hashmap.putLatestWriteForCell(r, commitTimestamp);
                        newLowWatermark = Math.max(removed, newLowWatermark);
                    }

                    lowWatermark = newLowWatermark;
                    LOG.trace("Setting new low Watermark to {}", newLowWatermark);
                    persistProc.persistLowWatermark(newLowWatermark);
                }
                persistProc.persistCommit(startTimestamp, commitTimestamp, c);
            } catch (IOException e) {
                LOG.error("Error committing", e);
            }
        } else { // add it to the aborted list
            persistProc.persistAbort(startTimestamp, isRetry, c);
        }

        return commitTimestamp;
    }
```

