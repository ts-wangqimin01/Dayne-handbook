# 简介
TiDB节点是TiDB集群中的SQL层，负责提供基于`mysql`协议的 Endpoint 供外部服务使用。同时负责将 SQL 翻译成 Key-Value 操作，将其转发给共用的分布式 Key-Value 存储层 TiKV，然后组装 TiKV 返回的结果，最终将查询结果返回给客户端。
TiDB是一种无状态节点，其节点本身不存储任何业务数据和集群数据。除本身启动的命令行选项外，绝大部分配置参数都以TiDB集群的系统变量形式进行管理，所以TiDB相关和集群相关的配置都存储在TiKV中。但也有一些节点级别的系统变量。

# SQL层
SQL数据查询向来都是一个极其复杂的过程。一条SQL语句会在TiDB节点中分析、优化、分发查询请求、汇总结果，并进行最后的计算再返回给客户端。
通常来说，判断TiDB集群是否有性能问题时，主要关注的是SQL的`duration`，即单次请求的返回时间。当确定存在性能问题的时候，会根据其他的metrics或者是TiDB dashboard中的分析工具对SQL语句的性能进行分析。
Slow query也是在TiDB层处理与分析。

# DDL
数据库中执行的`SQL`大概分以下两种：
 - `DDL`：`Data Definition Language` 用于定义数据库的结构及其对象，如表、视图、索引和过程。 DDL语句用于创建、更改和删除数据库对象，包括表、视图、索引和存储过程。
 - `DML`：`Data Manipulation Language` 用于操作数据库中的数据。

在TiDB中，使用在线异步的方式来执行DDL语句，当涉及到表中数据的操作时，会进行一次全表扫描。在大数据量的场景下，会消耗很多的性能，同时执行时间又很长。

TiDB会选择一个节点成为`DDL Owner`来管理和调度DDL语句的执行。多个DDL语句可以并发执行。在更高版本中（ >= 7.5 ），DDL 还可以分布式执行。

https://docs.pingcap.com/zh/tidb/stable/ddl-introduction/#ddl-%E8%AF%AD%E5%8F%A5%E7%9A%84%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86%E5%8F%8A%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5

# 节点管理
TiDB节点的增加与移除不需要特殊操作。部署并启动即可加入集群。关闭服务后即可在TiDB dashboard中将节点移除（Hide）。
但在运维的过程中仍然需要注意以下几点：
 - 在移除TiDB节点的时候需要注意节点上是否有在执行的DDL。
 - 移除节点时，需要考虑到连接到当前节点上的Application和`load balancer`。
 - TiDB虽然是无状态服务，但仍旧有一些场景会需要使用磁盘空间作为临时存储。所以选择节点的时候同样需要考虑到IO性能来应对一些极端情况。
