# 简介
PD (Placement Driver) 是 TiDB 集群的管理模块，同时也负责集群数据的实时调度。  
作为整个集群的核心，其内使用etcd作为数据的存储和管理核心。并以此数据为基础构建一个高性能的分布式调度器。

# 功能
PD作为整个集群的核心组件，提供一下功能：  
 - 元数据管理。PD会收集各个组件上报的数据，并为其他的组件提供数据索引服务。
 - 集群管理。根据输入的指令进行一些运维操作。例如节点的管理，副本的管理等。
 - 生成调度计划。PD会根据已经收集的信息，统计当前集群的状态，根据需要生成调度计划，由各自负责的组件拉取后执行。例如TiKV的数据均衡、热点数据均衡等。
 - 为各个组件提供TSO以进行event的追踪。为其他组件提供TSO是支持多版本并发控制 (Multi-Version Concurrency Control, MVCC)和事务的必要条件。
 - 提供dashboard可以对整个集群的状态、性能进行观察。并且提供多个性能追踪工具。

# TSO - TimeStamp Oracle
PD 承担着 TSO 时间戳分配器的角色，负责为集群内各组件分配时间戳。这些时间戳用于为事务和数据分配时间标记。  
转换为二进制后，前 46 位是物理时间戳，后 18 位是逻辑时间戳。  

# 运维操作
## 对PD的节点管理
可以直接通过 `pd-ctl` 对PD集群进行管理。
 - 踢出PD节点。 `member delete name $PD_NAME`
 - 切换Leader节点。 `member leader transfer $PD_NAME`
 - 设置dashboard地址。 `config set dashboard-address http://9.9.9.9:2379`

## 对TiKV节点的管理
可以直接通过 `pd-ctl` 控制整个集群中数据和存储节点的调度策略。包括以及不限于以下内容：
 - 踢出或取消踢出节点 `store delete $STORE_ID`  `store cancel-delete $STORE_ID`
 - 移除tombstone节点。 `store remove-tombstone`
 - 为节点分配权重。 `store weight $STORE_ID $LEADER_WEIGHT $REGION_WEIGHT`
 - 为节点分配label `store label $STORE_ID $KEY=$VAULE` `store label $STORE_ID $KEY --delete`
 - 调整节点之间传输数据的速率 `store limit [$STORE_ID|all] [add-peer|remove-peer] [INT]`

节点之间传输数据的速率不仅受到各个节点之间`add-peer`和`remove-peer`的限制，同样还有PD生成调度计划的限制

## 数据管理
使用 `pd-ctl` 直接对region进行操作， 需要注意的是以下行为可能判随着一定的风险性，操作不慎的情况下有可能造成数据问题：
 - 查询：
    - 查询region的元数据
    - 查询热点region
    - 查询状态异常的region
 - 删除。
 - 在某个节点上为某个region添加一个副本。
 - 合并。
 - 拆分。
 - 调度到其他节点。

## 副本策略管理
主要包括以下两大功能：
 - 根据TiKV节点的label，控制数据的副本分布。
 - 配置副本调度器，生成数据副本相关的调度计划：
    - 控制各种副本操作调度计划的生成速率。
    - 直接生成Leader调度策略。  
