# TiDB 中文技术文档

## 目录

+ TiDB 简介 & 整体架构
  - [TiDB 简介及核心 NewSQL 特性](#tidb-简介)
  - [TiDB 整体架构](#tidb-架构)
+ 安装 & 部署
  - [部署建议](op-guide/recommendation.md)
  - [Binary 下载](op-guide/binary-deployment.md#下载官方-binary)
  - [Binary 部署方案（推荐）](op-guide/binary-deployment.md)
  - [Docker 部署方案](op-guide/docker-deployment.md)
  - 配置 & 参数解释
+ 运维 & 监控
 - [整体监控框架概述](op-guide/monitor-overview.md)
 - [组件状态 API & 监控](op-guide/monitor.md)
+ SQL 兼容及对比
 - [TiDB SQL 文法](https://pingcap.github.io/sqlgram/)
 - [与 MySQL 兼容性对比](op-guide/mysql-compatibility.md)
+ [常用工具](https://github.com/pingcap/tidb-tools)
+ [常见问题与解答](./faq.md)
+ 故障诊断
+ 使用示例
+ 高级功能
  - [数据迁移方法](op-guide/migration.md)
  - 性能调优
  - [读取历史版本数据](op-guide/history-read.md)
+ 更多资源
  - [PingCAP 团队技术博客](http://www.pingcap.com/bloglist.html)

## TiDB 简介

TiDB 是 PingCAP 公司基于 Google Spanner / F1 论文实现的分布式 NewSQL 数据库。

TiDB 具备如下 NewSQL 核心特性
 * SQL支持 （TiDB 是 MySQL 兼容的）
 * 水平线性弹性扩展（容量、并发、吞吐量）
 * 分布式事务，数据强一致性保证
 * 故障自恢复的高可用 （auto failover）

TiDB 是传统的数据库中间件、数据库分库分表等 Sharding 方案非常优雅而理想的替换和解决方案。

## TiDB 整体架构

要深入了解 TiDB 的水平扩展和高可用特点，首先需要了解 TiDB 的整体架构。
![architecture](./op-guide/architecture.png)
TiDB 集群主要分为三个组件：
#### TiDB Server
TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。
TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。

#### PD Server
Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个： 一是存储集群的原信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID

PD 是一个集群，需要部署奇数个节点，一般线上推荐至少部署 3 个节点。

#### TiKV Server
TiKV Server 负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range （从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region 。TiKV 使用 Raft 协议做复制，保持数据的一致性和容灾。副本以 Region 位单位进行管理，不同节点上的多个 Region 构成一个 Raft Group，互为副本。
数据在多个 TiKV 之间的负载均衡由 PD 调度，这里也是以 Region 为单位进行调度。

### 水平扩展

无限水平扩展是 TiDB 的一大特点，这里说的水平扩展包括两方面：计算能力和存储能力。
TiDB Server 负责处理 SQL 请求，随着业务的增长，可以简单的添加 TiDB Server 节点，提高整体的处理能力，提供更高的吞吐。
TiKV 负责存储数据，随着数据量的增长，可以部署更多的 TiKV Server 节点解决数据 Scale 的问题。PD 会在 TiKV 节点之间以 Region 为单位做调度，将部分数据迁移到新加的节点上。
所以在业务的早期，可以只部署少量的服务实例（推荐至少部署 3 个 TiKV， 3 个 PD，2 个 TiDB），随着业务量的增长，按照需求添加 TiKV 或者 TiDB 实例。

### 高可用

高可用是 TiDB 的另一大特点，TiDB/TiKV/PD 这三个组件都能容忍部分实例失效，不影响整个集群的可用性。下面分别说明这三个组件的可用性、单个实例失效后的后果以及如何恢复。
#### TiDB
TiDB 是无状态的，推荐至少部署两个实例，前端通过负载均衡组件对外提供服务。当单个实例失效时，会影响正在这个实例上进行的 Session，从应用的角度看，会出现单次请求失败的情况，重新连接后即可继续获得服务。单个实例失效后，可以重启这个实例或者部署一个新的实例。

#### PD
PD 是一个集群，通过 Raft 协议保持数据的一致性，单个实例失效时，如果这个实例不是 Raft 的 leader，那么服务完全不受影响；如果这个实例是 Raft 的 leader，会重新选出新的 Raft leader，自动恢复服务。PD 在选举的过程中无法对外提供服务，这个时间大约是3秒钟。推荐至少部署三个 PD 实例，单个实例失效后，重启这个实例或者添加新的实例。

#### TiKV
TiKV 是一个集群，通过 Raft 协议保持数据的一致性（副本数量可配置，默认保存三副本），并通过 PD 做负载均衡调度。单个节点失效时，会影响这个节点上存储的所有 Region。对于 Region 中的 Leader 结点，会中断服务，等待重新选举；对于 Region 中的 Follower 节点，不会影响服务。当某个 TiKV 节点失效，并且在一段时间内（默认 10 分钟）无法恢复，PD 会将其上的数据迁移到其他的 TiKV 节点上。
