# 大数据管理技术

## HDFS

### 架构图

<img src="figures\HDFS.png" alt="HDFS" style="zoom:75%;" />

### 主要组件

#### NameNode

分类：

- Master NameNode：只有一个
- Secondary NameNode：定期合并文件系统镜像（FSImage）和操作日志（Editlog），推送给 Master
- Active NameNode
- Standby NameNode：当 Active NameNode 出现故障时，快速切换为 Active NameNode

作用：

- 管理HDFS的命名空间（NameSpace）
- 管理数据块（Data Block）的映射信息
- 配置副本放置策略
  - 副本1：Client的节点上
  - 副本2：与副本1不同机架的节点上
  - 副本3：与副本2同一机架的不同节点上
  - 其它副本：随机挑选

- 处理客户端（Client）读写请求
- 提供目录服务
- 提供灾难恢复的主节点
  - 为了提高效率，将元数据存放到内存里
  - 通过操作日志（Editlog）和定期的 Checkpoint 进行持久化


#### DataNode

本质：一个进程

特点：Slave，有多个

作用：

- 存储数据块，执行数据块读写操作

- 负责和 NameNode 以及 Client 的 RPC（Remote Procedure Call）通信

#### Client

作用：

- 管理并访问 HDFS
- 进行文件切分
- 与 NameNode 交互，获取文件位置信息

- 与 DataNode 交互，读写数据

### 适用场景

适合：批处理

不支持：查找、更新、插入、随机删除

## HBase

### 数据模型

#### 表（Table）

- 可以有多个
- 可能非常稀疏，很多 Cell 可以是空的

#### 行（Row）

- 每条数据的主键，是唯一的，方便快速查找
- 类型：Byte array 或 String

#### 列族（Column Family）

- 可以有多个
- 类型：可以为 String

#### 列（Column）

- 可以有多个
- 类型：可以为 String
- 名称是编码在 Cell 中的
- 不同的 Cell 可以拥有不同的列（Dynamic Column）

#### 时间戳（Timestamp）

- 可以由用户提供，无需以递增的顺序插入
- 每个 Cell 的时间戳是唯一的
- 类型：Long
- 默认值：系统时间戳

#### 值（Value）

- 类型：Byte array 或 String

### 存储结构

- 所有数据按照行键、列键、降序时间戳排序存储
- 不同列族的数据单独存储，同一个列族的不同列混合存储
- 表在行的方向上按大小分割为多个 Region：每个表一开始只有一个 Region，随着数据增多，Region 不断增大，当增大到一个阈值时，就会等分成两个新的 Region（Region Split），同理之后会有越来越多的 Region
- Region 是 HBase 中分布式存储和负载均衡的最小单元，不同的 Region 分布在不同的 Region Server 上
- Region 由一个或多个 Store 组成，每个 Store 保存一个列族，Store 是存储的最小单元
- 每个 Store 由一个 memStore 和零至多个 StoreFile 组成
- memStore 存储在内存中，StoreFile 存储在 HDFS 中
- 采用类似DBMS中稀疏数据的存储方式

### 架构图

<img src="figures\HBase.png" alt="HBase" style="zoom:75%;" />

### 主要组件

#### Master

作用：

- 处理 Schema 的更新请求
- 为 Region Server 分配 Region
- 负责 Region Server 的负载均衡
- 发现失效的 Region Server 并重新分配 Region
- 回收 HDFS 中的垃圾文件

故障：仍然可以访问 HBase

#### Region Server

作用：

- 维护 Region，进行 Region Split 操作
- 处理对 Region 的读写（IO）请求

故障：存在单独故障，需要十几分钟来重新启动

#### Client

作用：

- 包含和 ZooKeeper 交互来访问 HBase 的接口
- 维护一些缓存（Cache）来加快对 HBase 的访问，比如 Region 的位置信息

#### ZooKeeper

作用：

- Master 与 Region Server 启动时会向 ZooKeeper 注册
- 保证任何时候集群中只有一个 Master
- 使得 Master 故障不再是单点故障
- 实时监控 Region Server 的状态，将 Region server 的上线和下线信息实时通知给 Master
- 存储 Region 的寻址入口
- 存储 Schema，包括有哪些表（Table），每个表有哪些列族（Column Family）

### 适用场景

- 需要在大数据上进行操作，比如PB级的数据
- 需要进行高并发的操作，比如每秒进行上千次操作
- 需要对数据进行随机读操作或者随机写操作
- 数据读写操作均非常简单

### 与HDFS的关系

存储层搭建在 HDFS 上

## MapReduce

### 架构图 

<img src="figures\MapReduce.png" alt="MapReduce" style="zoom:75%;" />

1. Client 告诉 ResourceManager 中的 ApplicationManager 需要多少资源。

2. ApplicationManager 选择某个 NodeManager，让它启动 ApplicationMaster、命令和 jar 包。

3. ApplicationMaster 启动后，向 ApplicationManager 注册。

4. ApplicationMaster 向 ResourceScheduler 申请 MapTask 和 ReduceTask 的资源，ResourceScheduler 逐渐把拿到的资源告诉 ApplicationMaster。

5. ApplicationMaster 根据给它的资源和 NodeManager 通信，因为资源都是和 NodeManager 绑定的。

6. NodeManager 启动 Container、MapTask 和 ReduceTask。

7. Client 可以和 ApplicationMaster 通信，查询状态。

   MapTask 和 ReduceTask 需要和 ApplicationMaster 通信，汇报心跳。

8. 所有的 MapTask 和 ReduceTask 运行完成后，ApplicationMaster 告诉 ApplicationManager，ApplicationManager 把 ApplicationMaster 从集群里去掉。

### 主要组件

#### InputFormat

#### Mapper

#### Partitioner

#### RemoteReader

#### Sorter

#### Reducer

#### OutputFormat

### 原理

### 容错技术

ApplicationMaster 容错技术：

- 一旦运行失败，由 YARN 的 ResourceManager 负责重新启动
- 最高重启次数可由用户设置，默认是2次
- 一旦超过最高重启次数，则作业运行失败

MapTask 和 ReduceTask 容错技术：

- 周期性地向 ApplicationMaster 汇报心跳
- 一旦运行失败，ApplicationMaster 将为之重新申请资源并运行
- 最高重新运行次数可由用户设置，默认是4次
- 一旦超过最高重新运行次数，则作业运行失败

## Storm

### 架构图

<img src="D:\大数据管理技术\Final\figures\Storm.png" alt="Storm" style="zoom:75%;" />

### 主要组件

#### Nimbus

作用：

- 调度拓扑（Topology），将调度信息写入 ZooKeeper
- 处理拓扑的提交（Submit）、删除（Kill）、重平衡（Rebalance）等请求
- 提供文件的上传下载服务，如拓扑 jar 包
- 管理集群，提供查询集群或拓扑状态的 Thrift 接口
- 检查 Supervisor 和 Worker 的心跳信息，处理异常

故障：拓扑还可以正常运行，但如果 Worker 故障了，不能再重启这个 Worker

#### ZooKeeper

作用：

- 存储元数据
- 当作高可用 KV 使用，未使用监控（Watch）等高级 ZooKeeper 功能

| ZooKeeper 路径                        | 数据                     |
| ------------------------------------- | ------------------------ |
| /storm/storms/storm-id                | 拓扑基本信息             |
| /storm/assignments/storm-id           | 拓扑调度信息             |
| /storm/supervisors/superviser-id      | Supervisor 的心跳信息    |
| /storm/workerbeats/storm-id/node-port | Worker 的心跳信息        |
| /storm/errors/storm-id/component-id   | Spout 和 Bolt 的错误信息 |

#### Supervisor

本质：集群中的节点

作用：

- 定期在 ZooKeeper 更新心跳信息，默认为5s
- 根据 ZooKeeper 的调度信息，下载拓扑 jar 包并解压，启动一个或多个 Worker
- 检查 Worker 本地的心跳信息，处理 Worker 异常，启停 Worker

故障：拓扑还可以正常运行

#### Worker

本质：一个进程，资源分配的单位

作用：

- 定期在本地系统和 ZooKeeper 更新心跳信息
- 负责数据传输

  - 建立 Worker 之间的网络连接
  - 一对收发线程完成 Tuple 收发
- 启动一个或多个 Executor，处理一个特定的拓扑

#### Executor

本质：一个线程

作用：

- 创建一个特定的 Spout 或 Bolt 对象，启动一个或多个 Task

- 运行两个关键线程
  - 执行线程：执行 Spout.nextTuple() 和 Bolt.execute() 回调函数
  - 传输线程：将产生（Emit）的 Tuple 放到 Worker 传输队列
- 维护两个内存中的关键 DisrupterQueue

  - 从 ReceiveQueue 中获取收到的 Tuple 并处理

  - 产生的 Tuple 放到 TransferQueue

#### Task

本质：一个数据处理对象

## Spark

### 对象

名称：弹性分布式数据集（Resilient Distributed Datasets，RDD）

本质：由多个 Partition 构成、分布式的只读对象集合，可以看作一个数组

特点：

- 提供多种存储级别，可以存储在内存或磁盘中以便于重用，非常灵活
- 通过并行的 Transformation 操作构造
- 失效后通过 Lineage 血统原理自动重构

基本操作（Operater）：

- Transformation
  - 通过 Scala 集合或 Hadoop 数据集构造一个新的 RDD
  - 通过已有的 RDD 产生新的 RDD
  - 只会记录 RDD 的转化关系， 并不会触发计算，即惰性执行（Lazy Execution）
  - 例如：map，filter，groupByKey，reduceByKey
- Action 
  - 通过 RDD 计算得到一个或一组值，以保存（Save）或显示（Display）
  - 触发程序执行
  - 例如：count，reduce，saveAsTextFile

### 执行原理

1. RDDObject 构造操作符 DAG
2. DAGScheduler 将图分解成一系列的 Stage，每个 Stage 由若干个 Task 构成，将 TaskSet 提交到集群中
3. TaskScheduler 通过 ClusterManager 提交作业，并重新提交失败的 Task
4. Executer 中的线程执行 Task；其中的 BlockManager 存储数据块，提供数据块读写服务

### 容错技术 

基于 RDD 间的 Lineage 依赖图达成 Partition 粒度的 Lineage 容错机制

### 相比MapReduce的优势

- MapReduce 仅支持 Map 和 Reduce 两种操作，不适合流式计算、在线交互式计算和迭代计算，不够灵活；Spark 支持多种操作，可以进行流式计算、在线交互式计算和批处理计算，更灵活。
- MapReduce 处理效率低，其中 MapTask 的中间结果写到磁盘中，ReduceTask 的最终结果写到 HDFS 中，多个 MapReduce 操作之间通过 HDFS 交换数据，任务调度和启动开销较大，无法充分利用内存，MapTask 和 ReduceTask 均需要排序；Spark 处理效率高，比 MapReduce 快 10 到 100 倍，其中**内存计算引擎**提供缓存（Cache）机制来支持迭代计算或数据共享，减少数据的读写开销，**DAG引擎**减少中间结果写到 HDFS 的开销，使用**多线程池模型**来减少 Task 启动开销，避免不必要的排序操作，减少磁盘读写操作。

# 考试范围

1. HDFS、MapReduce、HBase、Storm的架构 (包括架构图和主要组件的介绍)
2. HDFS和HBase的适用场景，以及二者的关系
3. MapReduce的原理
4. HBase的数据模型（Data Model）
5. HBase的存储结构
6. Spark的transformation和action，以及惰性执行
7. Spark的RDD
8. Spark的执行原理
9. MapReduce和Spark的容错技术
10. Spark相比于MapReduce的优势