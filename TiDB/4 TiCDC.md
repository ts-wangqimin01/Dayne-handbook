# 简介
TiDB 增量数据同步工具，通过拉取上游 TiKV 的数据变更日志，TiCDC 可以将数据解析为有序的行级变更数据输出到下游。  
https://docs.pingcap.com/zh/tidb/stable/ticdc-overview/

# 工作机制
TiCDC通过连接上游PD集群，获取集群状态，在上游TiKV中订阅。由上游TiKV将数据的`change logs`发送给TiCDC集群进行处理。  
TiCDC通过注册在同一个PD集群中，使用相同的`cluster-id`来组成一个集群。集群通过PD在各节点共享changefeed，并选举一个Owner。  
Owner并不是一个节点角色，而是一个节点可包含的角色。当节点拥有owner的时候，他会负责管理集群中的changefeed，并执行同步DDL语句的任务。  
TiCDC中包含以下4种模块：
 - Puller：接收从TiKV推送过来的change logs保存在本地，如果发现有缺失的部分，则向TiKV请求增量扫描。
 - Sorter：将接收到的变更在 TiCDC 内按照时间戳进行升序排序，从而确保数据在表级别是满足一致性的。
 - Mounter：将变更按照对应的 Schema 信息转换成 TiCDC 可以处理的格式。如果启用了容灾最终一致性恢复（`eventual`），那么change log会在这个模块中解码为redo log，并发送到外部存储中。
 - Sink：将对应的变更应用到下游系统。

https://docs.pingcap.com/zh/tidb/stable/ticdc-architecture/

# Changefeed
Changefeed 是 TiCDC 中的单个同步任务。Changefeed 将一个 TiDB 集群中数张表的变更数据输出到一个指定的下游。TiCDC 集群可以运行和管理多个 Changefeed。
其中必要的要素为`Changefeed ID`和`下游sink-url`，其他的信息即使不提供也会使用默认值。
## Changefeed的状态：
### Normal
同步任务正常进行，`checkpoint-ts` 正常推进。处于这个状态的 changefeed 会阻塞 GC 推进。
### Stopped
同步任务停止，由于用户手动暂停 (pause) changefeed。处于这个状态的 changefeed 会阻挡 GC 推进。
### Warning
同步任务报错，由于某些可恢复的错误导致同步无法继续进行。处于这个状态的 changefeed 会不断重试，试图继续推进，直到状态转为 `Normal`。默认重试时间为 30 分钟（可以通过 `changefeed-error-stuck-duration` 调整），超过该时间，changefeed 会进入 `Failed` 状态。处于这个状态的 changefeed 会阻挡 GC 推进。
### Finished
同步任务完成，同步任务进度已经达到预设的 `TargetTs`。处于这个状态的 changefeed 不会阻挡 GC 推进。
### Failed
同步任务失败。处于这个状态的 changefeed 不会自动尝试恢复。为了让用户有足够的时间处理故障，处于这个状态的 changefeed 会阻塞 GC 推进，阻塞时长为 `gc-ttl` 所设置的值，其默认值为 24 小时。

## changefeed的lag
### Checkpoint
TiDB集群会根据`Region Leader`的时间计算出`Global ResolvedTS`。TiDB 集群确保 `Global ResolvedTS` 之前的事务都被提交了，或者可以认为这个时间戳之前不存在未提交的事务。  
在changefeed中也可以通过上下游的`Global ResolvedTS`来判断上游数据和下游数据存在多少延迟。  
在启用Syncpoint功能之后，TiCDC会保证上下游的快照一致性。Syncpoint功能会提供上游与下游满足快照一致性的TSO，并且可以对上下游的数据进行一致性校验。  
https://docs.pingcap.com/zh/tidb/stable/ticdc-upstream-downstream-check/ 
### Resolved ts lag
这是TiCDC从上游拉取数据时，在本地解析的进度延迟。当这个`Resolved ts lag`较高时，证明TiCDC存在性能瓶颈。
### Difference of resolvedTs and checkpointTs
当TiCDC性能满足要求，但下游存在瓶颈，接收数据慢的时候，difference lag 就会持续增高。此时需要查看下游集群是否存在性能异常。

## Dataflow
TiCDC有4种模块，对应的Changefeed就会存在4个阶段：
 - puller：从上游获取到的数据变成数量。
 - sorter：本地解析完成的数据。
 - mounter：解析完成后转码成功的数据。
 - sink：成功发送到下游的数据。
这4个模块的metrics可以用来判断TiCDC在哪个阶段存在性能瓶颈，同时也可以用来观测Changefeed是否运行正常。

## redo log
在启用了容灾最终一致性恢复（`eventual`）之后，TiCDC会把从上游拉取的数据转码成redo log，发送到外部存储（例如S3）。   
目的是为了在上游集群发生灾难并且下游数据在未完成同步的时候，可以使用外部存储将下游集群恢复到最近的事务版本。  
redo log由mounter模块转码后发送到外部存储。集群中的每个节点会发送各自接收到的数据，由Owner节点来维护redo log的meta data，并根据实际任务情况清理redo log。  
redo log的处理速度慢会阻塞数据向下游传输的速度，进而阻塞sorter模块。  
当下游存在问题，无法接收数据的时候同样会阻塞sink模块进而阻塞mounter和sorter。但由于TiCDC中同时存在多个mounter，并不会阻塞节点向外部存储发送数据。

# 运维操作
## changefeed的管理与查询
 - 查询有哪些changefeed：```./cdc cli --server "`hotsname -f`" changefeed list```
 - 暂停某个changefeed：```./cdc cli --server "`hotsname -f`" changefeed pause -c $CHANGEFEED_ID```
 - 恢复某个已经暂停的changefeed：```./cdc cli --server "`hotsname -f`" changefeed resume -c $CHANGEFEED_ID```
 - 查询某个changefeed的配置：```./cdc cli --server "`hotsname -f`" changefeed query -c $CHANGEFEED_ID```
 - 创建一个changefeed的配置：```./cdc cli --server "`hotsname -f`" changefeed create -c $CHANGEFEED_ID  --sink-url="" --config=*****```
 - 更新某个changefeed的配置：```./cdc cli --server "`hotsname -f`" changefeed update -c $CHANGEFEED_ID  --sink-url="" --config=*****```
 - 删除某个changefeed的配置：```./cdc cli --server "`hotsname -f`" changefeed remove -c $CHANGEFEED_ID  ```
## 查询当前TiCDC集群的节点和Owner
```./cdc cli --server "`hotsname -f`" capture list ```