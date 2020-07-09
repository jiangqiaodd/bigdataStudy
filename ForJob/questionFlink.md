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
#### (3) Flink组件栈

