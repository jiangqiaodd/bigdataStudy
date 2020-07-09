## Flink question
### 1 核心机制以及基础

#### （1）flink 基本介绍

- 计算框架和分布式处理引擎，对有界数据和无界数据流进行有状态处理，提供状态容错，资源管理等机制
- 提供多层次API抽象  StatefulStreamingProcessing DataStream/DataSet DataTable Sql
- 提供其他领域的资源库

#### （2）flink && spark

Flink 是标准的实时处理引擎，基于事件驱动。而 Spark Streaming 是微批（Micro-Batch）的模型
- 架构模型
```
    spark: Master Worker Driver Executor
    flink: Jobmanager taskMnager slot
```

- 任务调度
 ```
 Spark Streaming：连续不断的生成微小的数据批次，构建有向无环图DAG，Spark Streaming 会依次创建 DStreamGraph、JobGenerator、JobSchedule
 FLink 根据用户代码构建StreamGraph --> JobGraph  --> ExcutionGraph 
 JobManager根据ExecutionGraph 对job进行调度
 ```
 
- 时间机制
 ```
 Spark Streaming 支持的时间机制有限，只支持处理时间。
 Flink 支持了流处理程序在时间上的三个定义：处理时间、事件时间、注入时间。同时也支持 watermark 机制来处理滞后数据。
 ```
 
- 容错机制
 ```
 对于 Spark Streaming 任务，我们可以设置 checkpoint，然后假如发生故障并重启，我们可以从上次 checkpoint 之处恢复，但是这个行为只能使得数据不丢失，可能会重复处理，不能做到恰一次处理语义。
 Flink 则使用两阶段提交协议来解决这个问题。
```

#### （3）flink 组件栈

Flink 是一个分层架构的系统，每一层所包含的组件都提供了特定的抽象
- Deploy
    
    支持包括local、Standalone、Cluster(Yarn)、Cloud等多种部署模式
- Runtime

    支持 Flink 计算的核心实现，比如：支持分布式 Stream 处理、JobGraph到ExecutionGraph的映射、调度等等，为上层API层提供基础服务
- API & Libarary

    API 层主要实现了面向流（Stream）处理和批（Batch）处理API，其中面向流处理对应DataStream API
    
    API层之上构建的满足特定应用的实现计算框架，也分别对应于面向流处理和面向批处理两类。面向流处理支持：CEP（复杂事件处理）、基于SQL-like的操作（基于Table的关系操作）
    
#### (4) Flink 的运行必须依赖 Hadoop组件吗？

   Flink可以完全独立于Hadoop，在不依赖Hadoop组件下运行。
   但是做为大数据的基础设施，Hadoop体系是任何大数据框架都绕不过去的。
   Flink可以集成众多Hadooop 组件，例如Yarn、Hbase、HDFS等等。例如，
   - Flink可以和Yarn集成做资源调度，
   - 也可以读写HDFS，或者利用HDFS做检查点。

#### (5) Flink的基础编程模型

Flink程序映射到 streaming dataflows，由流（streams）和转换操作（transformation operators）组成。
Source --> transformation --> sink

#### (6) Flink集群有哪些角色？各自有什么作用？
Flink 程序在运行时主要有 TaskManager，JobManager，Client三种角色
- Client需要从用户提交的Flink程序配置中获取JobManager的地址，并建立到JobManager的连接，将Flink Job提交给JobManager (JobGraph)
- JobManager扮演着集群中的管理者 接收Flink Job 部署任务执行，协调检查点，Failover 故障恢复       
- TaskManager是实际负责执行计算，管理其所在节点上的资源信息，如内存、磁盘、网络，在启动的时候将资源的状态向JobManager汇报    
   
#### (7) 说说 Flink 资源管理中 Task Slot 的概念
TaskManager会将自己节点上管理的资源分为不同的Slot：固定大小的资源子集。这样就避免了不同Job的Task互相竞争内存资源    
TaskManager 是一个 JVM 进程，并会以独立的线程来执行一个task或多个subtask,这样执行线程的容器视为slot

#### (8) 说说 Flink 的常用算子
- Map：DataStream → DataStream
- Filter：过滤掉指定条件的数据
- KeyBy：按照指定的key进行分组
- Reduce：用来进行结果汇总合并
- Window：窗口函数，根据某些特性将每个key的数据进行分组

#### (9) 说说你知道的Flink分区策略？
分区策略是用来决定数据如何发送至下游。目前 Flink 支持了8中分区策略的实现。
- GlobalPartitioner  发到第一个
- ShufflePartitioner 随机发到下游哪一个
- RebalancePartitioner 循环发送到下游的每一个实例中进行
- RescalePartitioner 根据并行度循环发 
```
A B -> 1 2 3 4  那么就是A循环发到 1 2 B循环发到 3 4
A B C D -> 1 2  那么就是A B -> 1  C D -> 2
```
- BroadcastPartitioner广播分区会将上游数据输出到下游算子的**每个实例**中。适合于大数据集和小数据集做Jion的场景。
- ForwardPartitioner 用于将记录输出到下游本地的算子实例  要求算子并行度一样
- KeyGroupStreamPartitionerHash分区器。会将数据按 Key 的 Hash 值输出到下游算子实例中。
- CustomPartitionerWrapper用户自定义分区器。需要用户自己实现Partitioner接口，来定义自己的分区逻辑
#### (10)  Flink的并行度了解吗？Flink的并行度设置是怎样的？
- 操作算子层面(Operator Level)
- 执行环境层面(Execution Environment Level)
- 客户端层面(Client Level)
- 系统层面(System Level)
需要注意的优先级：算子层面>环境层面>客户端层面>系统层面。
#### (11) Flink的Slot和parallelism有什么区别？
slot是指taskmanager的并发执行能力 拥有的
parallelism是指taskmanager实际使用的并发能力。
假设我们把 parallelism.default 设置为1，那么9个 TaskSlot 只能用1个，有8个空闲

#### (12) Flink有没有重启策略？说说有哪几种？
- 固定延迟重启策略（Fixed Delay Restart Strategy） 配置最大重启次数，重启之间最小间隔 (超过次数后，作业失败)
- 故障率重启策略（Failure Rate Restart Strategy）  配置在给定时间，最多发生多少次故障（超过阈值后，作业失败）
- 没有重启策略（No Restart Strategy）
- Fallback重启策略（Fallback Restart Strategy）
默认重启策略是通过Flink的配置文件设置的flink-conf.yaml
定义策略的配置key为: restart-strategy。
如果未启用检查点，则使用“无重启”策略。
如果激活了检查点但未配置重启策略，则使用“固定延迟策略”：restart-strategy.fixed-delay.attempts: Integer.MAX_VALUE尝试重启。

#### (13) 用过Flink中的分布式缓存吗？如何使用？
Flink实现的分布式缓存和Hadoop有异曲同工之妙。
目的是在本地读取文件，并把他放在 taskmanager 节点中，防止task重复拉取。
```
val env = ExecutionEnvironment.getExecutionEnvironment

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
val input: DataSet[String] = ...
val result: DataSet[Integer] = input.map(new MyMapper())
...
env.execute()
```

## flink中的广播变量
_将数据广播出去，是完整的而没有被分开_
我们知道Flink是并行的，计算过程可能不在一个 Slot 中进行，
那么有一种情况即：当我们需要访问同一份数据。那么Flink中的广播变量就是为了解决这种情况。