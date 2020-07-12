## Flink question
### 1 核心机制以及基础

#### (1)flink 基本介绍

- 计算框架和分布式处理引擎，对有界数据和无界数据流进行有状态处理，提供状态容错，资源管理等机制
- 提供多层次API抽象  StatefulStreamingProcessing DataStream/DataSet DataTable Sql
- 提供其他领域的资源库

#### (2) flink && spark

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
#### (3) flink 组件栈
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

#### (14)flink中的广播变量

1. 出发点：某份数据需要完整性；我们知道Flink是并行的，计算过程可能不在一个 Slot 中进行，
那么有一种情况即：当我们需要访问同一份数据。那么Flink中的广播变量就是为了解决这种情况。
2. 使用
  - 初始化数据
  
    DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3)
  - 广播数据
    
    .withBroadcastSet(toBroadcast, "broadcastSetName");
  - 获取数据
    
    Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
3. 注意：
- 广播出去的变量存在于每个节点的内存中，所以这个数据集不能太大。因为广播出去的数据，会常驻内存，除非程序执行结束
- 广播变量在初始化广播出去以后不支持修改，这样才能保证每个节点的数据都是一致的。

#### (15) window
- Flink 支持两种划分窗口的方式，按照time和count
- flink支持窗口的两个重要属性（size和interval）    
    如果size=interval,那么就会形成tumbling-window(无重叠数据)  
    如果size>interval,那么就会形成sliding-window(有重叠数据)
- 组合形成四种窗口      
    time-tumbling-window 无重叠数据的时间窗口，设置方式举例：timeWindow(Time.seconds(5))  
    time-sliding-window 有重叠数据的时间窗口，设置方式举例：timeWindow(Time.seconds(5), Time.seconds(3))  
    count-tumbling-window无重叠数据的数量窗口，设置方式举例：countWindow(5)   
    count-sliding-window 有重叠数据的数量窗口，设置方式举例：countWindow(5,3) 
- 自定义window三要素  
还可以自定义window  dataStream.window(MyDisignWindow)
- Window Assigner：负责将元素分配到不同的window。
- Trigger 
Trigger即触发器，定义何时或什么情况下Fire一个window
- Evictor（可选） 
驱逐者，即保留上一window留下的某些元素。     
[](/srcFlink/windowWatermark.md)

#### (16) time
- EventTime  EventTime  
为基准来定义时间窗口将形成EventTimeWindow,要求消息本身就应该携带EventTime
- IngestionTime     
以 IngesingtTime 为基准来定义时间窗口将形成 IngestingTimeWindow,以 source 的systemTime为准    
- ProcessingTime    
以 ProcessingTime 基准来定义时间窗口将形成 ProcessingTimeWindow，以 operator 的systemTime 为准。

#### (17) 说说Flink中的状态存储？
- Flink在做计算的过程中经常需要存储中间状态，来避免数据丢失和状态恢复
- Flink提供了三种状态存储方式：MemoryStateBackend、FsStateBackend、RocksDBStateBackend

#### (18) Flink 中水印是什么概念，起到什么作用？
Watermark 是 Apache Flink 
- 为了处理 EventTime 窗口计算提出的一种机制, 
- 本质上是一种时间戳。
- 一般来讲Watermark经常和Window一起被用来处理乱序事件。
- watermark用于高速flink框架，不会有比这个事件更早的数据到来，可以关闭窗口了      
```
简单理解下： 当我们基于eventTime时，如果数据产生时间比较早，但是由于网络或者其他原因
到达flink处理比较晚，此时由于按照eventTime时间格式 作为window划分
- 开着的window判定这个event数据的eventtime不在window范围内，不采集
- 旧的window已经关闭了，会造成这种延迟数据的丢失；
```
- 举例说明  
背景：生成时间为 13  13 16 依次有序到达      
one: processTime window 理想情况， 准时到达flink端        
![dd](/src/resource/watermarker_one.png)    
two: processTime 当event延迟到达：有一个13的数据延迟了6秒19才到       
![dd](/src/resource/watermarker_two.png)        
此时，根据processTime的Window 会将其中一个13分到 本不属于他的窗口     
three: eventTime window 当延迟到达时，由于13的数据在19时刻到来时窗口[10-15]已经关了，所以没有进入第一个窗口     
![dd](/src/resource/watermarker_three.png)      
four: eventTime window + watermark 使用watermark表示在这个时间戳之前的数据已经都到了，watermark = nowtimestamp - 5 ,会给于 window 5秒的等待缓冲       
![dd](/src/resource/watermarker_three.png)          
13的数据虽然在19秒到达，小于第一个窗口max_time + delay(5) 因此存放到第一个window中

- 使用watermark的方式    
（1）配置AllowedLateness    
（2）对数据流设置 自定义的TimestampAndWatermark     
两种WaterMark算子实现：    
- TimestampsAndPeriodicWatermarksOperator   定时产生插入watermark
- TimestampsAndPunctuatedWatermarksOperator 每条消息一个
```
senv.socketTextStream("localhost", 9999)
                .assignTimestampsAndWatermarks(new TimestampExtractor)
```
```                
class TimestampExtractor extends AssignerWithPeriodicWatermarks[String] with Serializable {
  override def extractTimestamp(e: String, prevElementTimestamp: Long) = {
    e.split(",")(1).toLong 
  }
  override def getCurrentWatermark(): Watermark = { 
      new Watermark(System.currentTimeMillis - 5000)
  }
}
```
- 注意    
（1）窗口中的消息不会根据事件时间进行排序


#### (19) Flink Table & SQL 熟悉吗？TableEnvironment这个类有什么作用
- 在内部catalog中注册表
- 注册外部catalog
- 执行SQL查询
- 注册用户定义（标量，表或聚合）函数
- 将DataStream或DataSet转换为表
- 持有对ExecutionEnvironment或StreamExecutionEnvironment的引用

#### (20) Flink SQL的实现原理是什么？ 是如何实现 SQL 解析的呢？
Flink 的SQL解析是基于Apache Calcite这个开源框架。    
[to do ](to do)

### 2 部分内容进阶篇
#### 2.1 Flink是如何支持批流一体的？
Flink的开发者认为批处理是流处理的一种特殊情况。
批处理是有限的流处理。Flink 使用一个流处理引擎支持了DataSet API 和 DataStream API。

#### 2.2 Flink是如何做到高效的数据交换的？ 
在一个Flink Job中，数据需要在不同的task中进行交换，整个数据交换是有 TaskManager 负责的，
- TaskManager 的网络组件首先从缓冲buffer中收集records，然后再发送。
- Records 并不是一个一个被发送的，而是积累一个批次再发送，batch 技术可以更加高效的利用网络资源。
         
#### 2.3 Flink是如何做容错的？
 Flink 实现容错主要靠强大的CheckPoint机制和State机制。
 - Checkpoint 负责定时制作分布式快照、对程序中的状态进行备份；
 - State 用来存储计算过程中的中间状态。
 
 #### 2.4 Flink 分布式快照的原理是什么？
  (1) Flink的分布式快照是根据Chandy-Lamport算法量身定做的     
  Chandy-Lamport 算法具体的工作流程
  - Initiating a snapshot: 也就是开始创建 snapshot，可以由系统中的任意一个进程发起
  - Propagating a snapshot: 系统中其他进程开始逐个创建 snapshot 的过程
  - Terminating a snapshot: 算法结束条件
  (2) flink checkpoint 执行流程
  核心思想是在 input source 端插入 barrier，控制 barrier 的同步来实现 snapshot 的备份和 exactly-once 语义。  
  checkpoint管理器，CheckpointCoordinator 由JobManager为创建，全权负责本job的快照制作。
  - CheckpointCoordinator周期性的向该流应用的所有source算子发送barrier。
  - 某个source算子收到一个barrier时，便暂停数据处理过程，然后将自己的当前状态制作成快照，并保存到指定的持久化存储中，最后向CheckpointCoordinator报告自己快照制作情况，同时向自身所有下游算子广播该barrier，恢复数据处理
  - 下游算子收到barrier之后，会暂停自己的数据处理过程，然后将自身的相关状态制作成快照，并保存到指定的持久化存储中，最后向CheckpointCoordinator报告自身快照情况[备份数据的地址（state handle）]，同时向自身所有下游算子广播该barrier，恢复数据处理
  - 每个算子按照步骤3不断制作快照并向下游广播，直到最后barrier传递到sink算子，快照制作完成
  - 当CheckpointCoordinator收到所有算子的报告之后，认为该周期的快照制作成功, 整个 StateHandle 封装成 completed CheckPoint Meta,写入到hdfs; 否则，如果在规定的时间内没有收到所有算子的报告，则认为本周期快照制作失败;     
  (3) checkpoint中保存的是什么信息(准确滴将是 snapshot)
  根据不同算子的snapshotState方法，存放ListState    
  (4) barrier 对齐
  一个Operator的流入端有多个入口时，当收到其中一个入口的barrier n，需要暂停来自任何流的数据记录，直到它从其他所有输入接收到barrier n为止。否则，它会混合属于快照n的记录和属于快照n + 1的记录；    
  接收到barrier n的流暂时被搁置。从这些流接收的记录不会被处理，而是放入输入缓冲区。     
  ***这里就是barrier对齐，等到所有barrier都来了之后，才进行checkpoint，然后下发barrier，继续处理数据    
  - barrier不对齐  
    barrier不对齐就是指当还有其他流的barrier还没到达时，为了不影响性能，也不用理会，直接处理barrier之后的数据。等到所有流的barrier的都到达后，就可以对该Operator做CheckPoint了
  - barrier对齐和不对齐的影响和区别
    ```
    * Exactly Once时必须barrier对齐，如果barrier不对齐就变成了At Least Once
    * barrier不对齐，可能造成从checkpoint恢复时，造成重复消费，因为恢复时，operator存放的状态可能是处理过上一级barrier之后的状态，包含一部分冗余的数据
    ```
  - barrier对齐出现在什么时候
  ```
  ** 首先设置了Flink的CheckPoint语义是：Exactly Once
  ** Operator实例必须有多个输入流才会出现barrier对齐
  ```
 #### 2.5 Flink 是如何保证Exactly-once语义的？       
  (1) 概述 flink的exactly-once我们可以细分为 
  - flink内部的exactly once : 如上面讲到的barrier 对齐实现消费exactly once
  - 端到端的exactly once    
 
  (2) Flink通过实现两阶段提交和状态保存来实现端到端的一致性语义。分为以下几个步骤：
  - 开始事务（beginTransaction）创建一个临时文件夹，来写把数据写入到这个文件夹里面
  - 预提交（preCommit）将内存中缓存的数据写入文件并关闭
  - 正式提交（commit）将之前写完的临时文件放入目标目录下。这代表着最终的数据会有一些延迟
  - 丢弃（abort）丢弃临时文件
  若失败发生在预提交成功后，正式提交前。可以根据状态来提交预提交的数据，也可删除预提交的数据。
  这四个步骤，抽象成TwoPhaseCommitSinkFunction类，而目前kafka的高版本的sink是继承了这个类的，通过开启exactly once就能够实现事务保证exactlyonce
  ```
  端到端的 exactly once 举例 kafka source --> window --> kafka sink
  作业必须在一次事务中将缓存的数据全部写入kafka。一次commit会提交两个checkpoint之间所有的数据。
  
  A pre-commit阶段起始于一次快照的开始，即master节点将checkpoint的barrier注入source端，barrier随着数据向下流动直到sink端。
    barrier每到一个算子，都会触发算子做本地快照，当状态涉及到外部系统时，需要外部系统支持事务操作来配合flink实现2PC协议
    【2pc: 两阶段提交，预提交 正式提交 当预提交发生问题时，取消这次行为】
  B 当所有的算子都做完了本地快照并且回复到master节点时，pre-commit阶段才算结束。这个时候，checkpoint已经成功，并且包含了外部系统的状态。如果作业失败，可以进行恢复。
  C 2PC的commit阶段：source节点和window节点没有外部状态，所以这时它们不需要做任何操作。而对于sink节点，需要commit这次事务，将数据写到外部系统。 
  ```   
  ![2PC提交pre-commit](/src/resource/exacutly_one.png)    
  ![2PC提交commit](/src/resource/exacutly_two.png)
    
  #### 2.6 flink的内存管理
  
  #####  (1) 概述 
 - Flink 并不是将大量对象存在堆上，而是将对象都序列化到一个预分配的内存块上(MemorySegment)。它代表了一段固定长度的内存（默认大小为 32KB），也是 Flink 中最小的内存分配单元  java.nio.ByteBuffer 可分配在堆内new byte[] 和堆外DirectByteBuffer        
 - 此外，Flink大量的使用了堆外内存。如果需要处理的数据超出了内存限制，则会将部分数据存储到硬盘上。
 - Flink 为了直接操作二进制数据实现了自己的序列化框架      
  
  ##### (2) 理论上TaskManger的内存管理分为三部分：
  
  - Network Buffers：这个是在TaskManager启动的时候分配的，这是一组用于缓存网络数据的内存，每个块是32K，默认分配2048个，可以通过“taskmanager.network.numberOfBuffers”修改
  - Memory Manage pool：大量的Memory Segment块，由MemoryManager管理，用于运行时的算法（Sort/Join/Shuffle等）向这个内存池申请 MemorySegment，将序列化后的数据存于其中，使用完后释放回内存池。池子占了堆内存（taskmanager进程分配的内存）的 70% 的大小
  - User Code，这部分的内存是留给用户代码以及 TaskManager 的数据结构使用的。因为这些数据结构一般都很小，所以基本上这些内存都是给用户代码使用的。从GC的角度来看，可以把这里看成的新生代，也就是说这里主要都是由用户代码生成的短期对象。       
  Flink 采用类似 DBMS 的 sort 和 join 算法，直接操作二进制数据，从而使序列化/反序列化带来的开销达到最小；  
  ##### (3) Flink 积极的内存管理以及直接操作二进制数据优点
  - 减少GC压力。显而易见，因为所有常驻型数据都以二进制的形式存在 Flink 的MemoryManager中，这些MemorySegment一直呆在老年代而不会被GC回收(甚至可以放在堆外，进一步降低JVM GC耗费)    
  - 避免了OOM。所有的运行时数据结构和算法只能通过内存池申请内存，保证了其使用的内存大小是固定的 不会因为运行时数据结构和算法而发生OOM； 在内存吃紧的情况下，算法（sort/join等）会高效地将一大批内存块写到磁盘，之后再读回来。因此，OutOfMemoryErrors可以有效地被避免
  - 节省内存空间。Java 对象在存储上有很多额外的消耗（如上一节所谈）。如果只存储实际数据的二进制内容
  - 高效的二进制操作 & 缓存友好的计算。二进制数据以定义好的格式存储，可以高效地比较与操作  
  
  ##### (4) 为什么还要引入堆外内存       
   - 启动超大内存（上百GB）的JVM需要很长时间，GC停留时间也会很长（分钟级）。使用堆外内存的话，可以极大地减小堆内存
   - 高效的 IO 操作。堆外内存在写磁盘或网络传输时是 zero-copy，而堆内存的话，至少需要 copy 一次。
   - 堆外内存是进程间共享的。也就是说，即使JVM进程崩溃也不会丢失数据。这可以用来做故障恢复（Flink暂时没有利用起这个，不过未来很可能会去做）。
   ##### (5) flink怎么使用堆外内存的
   Flink 将原来的 MemorySegment 变成了抽象类，并生成了两个子类。
   - HeapMemorySegment 和前者是用来分配堆内存的，     
      代码中所有的短生命周期和长生命周期的MemorySegment都实例化其中一个子类，另一个子类根本没有实例化过（使用工厂模式来控制）。那么运行一段时间后，JIT 会意识到所有调用的方法都是确定的，然后会做优化。
   - HybridMemorySegment。后者是用来分配堆外内存和堆内存的    
   Flink 优雅地实现了一份代码能同时操作堆和堆外内存。这主要归功于 sun.misc.Unsafe提供的一系列方法，如getLong方法
      ```
      sun.misc.Unsafe.getLong(Object reference, long offset)
      ** 如果reference不为空，则会取该对象的地址，加上后面的offset，从相对地址处取出8字节并得到 long。这对应了堆内存的场景。
      ** 如果reference为空，则offset就是要操作的绝对地址，从该地址处取出数据。这对应了堆外内存的场景。
      ```
   ##### (6) 内存划分
   在设置完taskManager内存之后相当于向yarn申请这么大内存的container，然后flink内部的内存大部分是由flink框架管理，在启动container之前就会 预先计算各个内存块的大小。       
      
   - 先预留出一块【总的 25% 】（最少60M），得到剩下可分配的JVM内存 javaMemorySizeMB = containerMemoryMB - cutoff;
   - 计算TaskManagerServices.calculateHeapSizeMB(javaMemorySizeMB, config) 得到堆内存  
    计算规则：
        默认开启了堆外内存： 那么将 [ jvmMemory - networkBuffer(32k * 2048=64M) ] * 70% 视为堆外 memoryManager管理（预设定，实际上更大些），
                                    [ jvmMemory - networkBuffer(32k * 2048=64M) ] * 70% 视为堆内  真正的heapMemory
   - 堆内存以下的为为堆外 containerMemoryMB - heapSizeMB  
   所以实际上，堆外： networkBuffer   预留的25%   + 0.75*0.7 差不多这些
               堆内： （总的-25%-networkbuffer）*30%  21%左右
   ```
   // 默认值0.25f
   final float memoryCutoffRatio = config.getFloat(ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_RATIO);
   // 最少预留大小默认600MB
   final int minCutoff = config.getInteger(ResourceManagerOptions.CONTAINERIZED_HEAP_CUTOFF_MIN);
             
   // 先减去一块内存预留给jvm
   long cutoff = (long) (containerMemoryMB * memoryCutoffRatio);
   if (cutoff < minCutoff) {
       cutoff = minCutoff;
   }
   final long javaMemorySizeMB = containerMemoryMB - cutoff;
   
   // (2) split the remaining Java memory between heap and off-heap
   final long heapSizeMB = TaskManagerServices.calculateHeapSizeMB(javaMemorySizeMB, config);
   // use the cut-off memory for off-heap (that was its intention)
   // 计算得到堆外内存后，总内存减去得到堆内的大小
   final long offHeapSize = javaMemorySizeMB == heapSizeMB ? -1L : containerMemoryMB - heapSizeMB;
   ```
   ```
   // 默认线上是开启堆外内存的，为了数据交换的过程只使用堆外内存，gc友好
   if (useOffHeap) {
       // subtract the Java memory used for network buffers
       final long networkBufMB = calculateNetworkBufferMemory(totalJavaMemorySize, config) >> 20; // bytes to megabytes
       final long remainingJavaMemorySizeMB = totalJavaMemorySizeMB - networkBufMB;
   
       long offHeapSize = config.getLong(TaskManagerOptions.MANAGED_MEMORY_SIZE);
   
       if (offHeapSize <= 0) {
           // calculate off-heap section via fraction
           // 将划去networkBuffer大小*一个堆外的系数（默认是0.7）得到其他的堆外内存
           double fraction = config.getFloat(TaskManagerOptions.MANAGED_MEMORY_FRACTION);
           offHeapSize = (long) (fraction * remainingJavaMemorySizeMB);
       }
   
       TaskManagerServicesConfiguration
           .checkConfigParameter(offHeapSize < remainingJavaMemorySizeMB, offHeapSize,
               TaskManagerOptions.MANAGED_MEMORY_SIZE.key(),
               "Managed memory size too large for " + networkBufMB +
                   " MB network buffer memory and a total of " + totalJavaMemorySizeMB +
                   " MB JVM memory");
   
       heapSizeMB = remainingJavaMemorySizeMB - offHeapSize;
   } else {
       heapSizeMB = totalJavaMemorySizeMB;
   }
   ```   
   demo
   ```
   一个启动时设置TaskManager内存大小为1024MB
   
   1024MB - (1024 * 0.2 < 600MB) -> 600MB  = 424MB (cutoff)
   424MB - (424MB * 0.1 < 64MB) -> 64MB = 360MB (networkbuffer)
   360MB * (1 - 0.7) = 108MB -> （onHeap）
   1024MB - 108MB = 916MB （maxDirectMemory）
   最终启动命令：
   yarn      46218  46212  1 Jan08 ?        00:17:50
   /home/yarn/java-current/bin/java 
   -Xms109m -Xmx109m -XX:MaxDirectMemorySize=915m -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
   -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=2 -XX:GCLogFileSize=512M 
   -Xloggc:/data1/hadoopdata/nodemanager/logdir/application_1545981373722_0172/container_e194_1545981373722_0172_01_000005/taskmanager_gc.log 
   -XX:+UseConcMarkSweepGC 
   -XX:CMSInitiatingOccupancyFraction=75 
   -XX:+UseCMSInitiatingOccupancyOnly 
   -XX:+AlwaysPreTouch -server 
   -XX:+HeapDumpOnOutOfMemoryError 
   -Dlog.file=/data1/hadoopdata/nodemanager/logdir/application_1545981373722_0172/container_e194_1545981373722_0172_01_000005/taskmanager.log 
   -Dlogback.configurationFile=file:./logback.xml 
   -Dlog4j.configuration=file:./log4j.properties org.apache.flink.yarn.YarnTaskManager 
   --configDir 
   ```
  #### 2.7 flink的序列化
  Java本身自带的序列化和反序列化的功能，但是辅助信息占用空间比较大，在序列化对象时记录了过多的类信息
  Apache Flink摒弃了Java原生的序列化方法，以独特的方式处理数据类型和序列化，包含自己的类型描述符（TypeInformatio），泛型类型提取和类型序列化框架（TypeSerializer）
  ##### (1) 定制序列化工具的优点
  - 由于数据集对象的类型固定，对于数据集可以只保存一份对象Schema信息，节省大量的存储空间
  - 对于固定大小的类型，可通过固定的偏移位置存取，可以直接通过偏移量，只是反序列化特定的对象成员变量，不需要反序列化整个对象
  
  
  ##### (2)类型信息由 TypeInformation  支持这些类型信息的序列化类
  - BasicTypeInfo: 任意Java 基本类型（装箱的）或 String 类型。
  - BasicArrayTypeInfo: 任意Java基本类型数组（装箱的）或 String 数组。
  - WritableTypeInfo: 任意 Hadoop Writable 接口的实现类。
  - TupleTypeInfo: 任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现。
  - CaseClassTypeInfo: 任意的 Scala CaseClass(包括 Scala tuples)。
  - PojoTypeInfo: 任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法。
  - GenericTypeInfo: 任意无法匹配之前几种类型的类。
  
  每个TypeInformation中，都包含了serializer，类型会自动通过serializer进行序列化，然后用Java Unsafe接口写入MemorySegments
  对于可以用作key的数据类型，Flink还同时自动生成TypeComparator
  序列化和比较时会委托给对应的serializers和comparators     
  ##### (3) Flink 如何直接操作二进制数据
    是将序列化数据本身和 指向序列化数据 的地址 分开存放的，这样，不用移动大量序列化对象
    以sort为例：
  - 首先，Flink 会从 MemoryManager 中申请一批 MemorySegment，我们把这批 MemorySegment 称作 sort buffer，用来存放排序的数据。
    分为两块 一个区域是用来存放所有对象完整的二进制数据。另一个区域用来存放指向完整二进制数据的指针以及定长的序列化后的key（key+pointer）
  - 排序的关键是比大小和交换。Flink 中，会先用 key 比大小，这样就可以直接用二进制的key比较而不需要反序列化出整个对象。因为key是定长的，所以如果key相同（或者没有提供二进制key），那就必须将真实的二进制数据反序列化出来，然后再做比较。
  - 之后，只需要交换key+pointer就可以达到排序的效果，真实的数据不用移动
  - 访问排序后的数据，可以沿着排好序的key+pointer区域顺序访问，通过pointer找到对应的真实数据，并写到内存或外部      
  
  
  
### 3 常见面试题
#### 3.1 Flink中的Window出现了数据倾斜，你有什么解决办法？
window产生数据倾斜指的是数据在不同的窗口内堆积的数据量相差过多。本质上产生这种情况的原因是数据源头发送的数据量速度不同导致的。出现这种情况一般通过两种方式来解决：    
- 在数据进入窗口前做预聚合  
    可以在window后跟一个reduce方法，在窗口触发前采用该方法进行聚合操作（类似于MapReduce 中  map端combiner预处理思路）。如使用 flink 的 aggregate 算子 
    ```
    
    ```
- 重新设计窗口聚合的key      
  将key进行扩展，扩展成自定义的负载数，即，将原始的key封装后新的带负载数的key，进行逻辑处理，
  然后再对新key的计算结果进行聚合，聚合成原始逻辑的结果。 
  ```
  1.首先将key打散，我们加入将key转化为 key-随机数 ,保证数据散列    
   val dataStream: DataStream[(String, Long)] = typeAndData
           .map(x => (x._1 + "-" + scala.util.Random.nextInt(100), x._2))
  2.对打散后的数据进行聚合统计，这时我们会得到数据比如 : (key1-12,1),(key1-13,19),(key1-1,20),(key2-123,11),(key2-123,10)    
   val keyByAgg: DataStream[DataJast] = dataStream.keyBy(_._1)
        .timeWindow(Time.seconds(10))
        .aggregate(new CountAggregate())
   
   
  //计算keyby后，每个Window中的数据总和
  class CountAggregate extends AggregateFunction[(String, Long),DataJast, DataJast] {
 
    override def createAccumulator(): DataJast = {
      println("初始化")
      DataJast(null,0)
    }
 
    override def add(value: (String, Long), accumulator: DataJast): DataJast = {
      if(accumulator.key==null){
        printf("第一次加载,key:%s,value:%d\n",value._1,value._2)
        DataJast(value._1,value._2)
      }else{
        printf("数据累加,key:%s,value:%d\n",value._1,accumulator.count+value._2)
        DataJast(value._1,accumulator.count + value._2)
      }
    }
 
    override def getResult(accumulator: DataJast): DataJast = {
      println("返回结果："+accumulator)
      accumulator
    }
 
    override def merge(a: DataJast, b: DataJast): DataJast = {
      DataJast(a.key,a.count+b.count)
    }
     
  3.将散列key还原成我们之前传入的key，这时我们的到数据是聚合统计后的结果，不是最初的原数据
  4.二次keyby进行结果统计，输出到addSink    
  还原key，再次keyBy
   val result: DataStream[DataJast] = keyByAgg.map(data => {
         val newKey: String = data.key.substring(0, data.key.indexOf("-"))
         println(newKey)
         DataJast(newKey, data.count)
       }).keyBy(_.key)
         .process(new MyProcessFunction())
  ```   
  ![重新设计窗口聚合的key](/src/resource/qinxie2.png)
  
#### 3.2 Flink中在使用聚合函数 GroupBy、Distinct、KeyBy 等函数时出现数据热点该如何解决？
举例： 北京上海的业务数量相比于其他城市更多，这种数据量大的就是热点数据，容易造成倾斜     
数据倾斜和数据热点是所有大数据框架绕不过去的问题。处理这类问题主要从3个方面入手：
- 在业务侧避免将 热点数据和 其他数据混在一起操作
- 把热key进行拆分，比如上个例子中的北京和上海，可以把北京和上海按照地区进行拆分聚合。
- 参数设置 Flink 1.9.0 SQL(Blink Planner) 性能优化中一项重要的改进就是
升级了微批模型，即 MiniBatch。原理是缓存一定的数据后再触发处理，以减少对State的访问，从而提升吞吐和减少数据的输出量。

#### 3.3 Flink任务延迟高如何处理？ Flink的反压以及如何处理反压
##### A 延迟处理
在Flink的后台任务管理中，我们可以看到Flink的哪个算子和task出现了反压，处理方案
- 资源调优 
    更多内存  更多cpu（taskmanger的slotgroup数目） 配置更多并行度
- 算子调优  
    operator的可用并发数、state的设置、checkpoint的设置
##### B 处理反压
Flink 内部是基于 producer-consumer 模型来进行消息传递的    
Flink 使用了高效有界的分布式阻塞队列，就像 Java 通用的阻塞队列（BlockingQueue）一样。下游消费者消费变慢，上游就会受到阻塞。
##### C flink 反压和 storm 反压
- Storm 是通过监控 Bolt 中的接收队列负载情况，如果超过高水位值就会将反压信息写到 Zookeeper ，Zookeeper 上的 watch 会通知该拓扑的所有 Worker 都进入反压状态，最后 Spout 停止发送 tuple。   
- Flink中的反压使用了高效有界的分布式阻塞队列，下游消费变慢会导致发送端阻塞。

#### 3.4 Flink Job的提交流程
#### 3.5 FLink 三层图结构    
##### （1) streamGraph jobGraph executionGraph
- StreamGraph     
最接近代码所表达的逻辑层面的计算拓扑结构，
按照用户代码的执行顺序向StreamExecutionEnvironment添加
StreamTransformation构成流式图。 StramNode
- JobGraph      
  从StreamGraph生成，将可以串联合并的节点进行合并，operator chain
  设置节点之间的边，安排资源共享slot槽位和放置相关联的节点，
  上传任务所需的文件，设置检查点配置等。
  相当于经过部分初始化和优化处理的任务图。 JobVectex
- ExecutionGraph    
  由JobGraph转换而来，包含了任务具体执行所需的内容，是最贴近底层实现的执行图。 
##### (2) operator chain
Flink会尽可能地将operator的subtask链接（chain）在一起形成task。每个task在一个线程中执行    
operator chain优点
- 减少减少线程之间的切换
- 减少消息的序列化/反序列化
- 减少数据在缓冲区的交换   
==》减少了延迟的同时提高整体的吞吐量     

operator chain在一起的的条件
- 上下游的并行度一致 
- 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
- 上下游节点都在同一个 slot group 中（下面会解释 slot group）     
- 用户没有禁用 chain 
- 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS） 
- 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD） 
#### 3.6 Master 
整个集群的一些协调工作,Flink 中 Master 节点主要包含三大组件：
- Flink Resource Manager、
- Flink Dispatcher 
- Flink Jobmanager 以及为每个运行的 Job 创建一个 JobManager 服务      
可以任务Master是flink集群的概念， JobMaster/JobManager是单个作业管理者的角色      

集群的 Master 节点的工作范围与 JobManager 的工作范围还是有所不同的，
- Master 节点的其中一项工作职责就是为每个提交的作业创建一个 JobManager 对象
- JobManager 任务的分发和取消，处理某个具体作业相关协调工作 task 的调度、Checkpoint 的触发及失败恢复， 心跳检测等
##### (1) Dispatcher:   
负责接收用户提供的作业，并且负责为这个新提交的作业拉起一个新的 JobManager 服务；
##### (2) ResourceManager:  
负责资源的管理，在整个 Flink 集群中只有一个 ResourceManager，资源相关的内容都由这个服务负责；
##### (3) JobManager:   
负责管理具体某个作业的执行，在一个 Flink 集群中可能有多个作业同时执行，每个作业都会有自己的 JobManager 服务

##### (4) 作业提交流程        
A. client端代码生成stramGraph 优化为jobGraph在这个过程中，
它会进行一些检查或优化相关的工作
（比如：检查配置，把可以 Chain 在一起算子 Chain 在一起）。    
B. 然后，Client 再将生成的 JobGraph 提交到集群中执行。
此时有两种情况（对于两种不同类型的集群）：
- 类似于 Standalone 这种 Session 模式（对于 YARN 模式来说），这种情况下 Client 可以直接与 Dispatcher 建立连接并提交作业；
- 是 Per-Job 模式，这种情况下 Client 首先向资源管理系统 （如 Yarn）申请资源来启动 ApplicationMaster，然后再向 ApplicationMaster 中的 Dispatcher 提交作业。      
按照本地集群还是yarn集群分 
- 本地环境下，MiniCluster完成了大部分任务，直接把任务委派给了MiniDispatcher；    
- 远程环境下，请求发到集群上之后，必然有个handler去处理，在这里是JobSubmitHandler。这个类接手了请求后，委派StandaloneDispatcher启动job，        
C. Dispatcher 会首先启动一个 JobManager 服务，
然后 JobManager 会向 ResourceManager 申请资源来启动作业中具体的任务。
ResourceManager 选择到空闲的 Slot （Flink 架构-基本概念）之后，就会通知相应的 TM 将该 Slot 分配给指定的 JobManager

#### 3.7 JobManger
##### （1）   负责自己作业相关的协调工作，包括：
- 向 ResourceManager 申请 Slot 资源来调度相应的 task 任务、
- 定时触发作业的 checkpoint 和手动 savepoint 的触发、
- 以及作业的容错恢复
- taskmanager的心跳信息
- 任务的请求部署调度和取消等

##### （2） 组件    
- LegacyScheduler:  
ExecutionGraph 相关的调度都是在这里实现的，它类似更深层的抽象，
封装了 ExecutionGraph 和 BackPressureStatsTracker，
JobMaster 不直接去调用 ExecutionGraph 和 BaaohckPressureStatsTracker 的相关方法，
都是通过 LegacyScheduler 间接去调用；
LegacyScheduler包含功能块： Blobwriter checkpointCoordinator 等
SlotPool:   
它是 JobMaster 管理其 slot 的服务，
它负责向 RM 申请/释放 slot 资源，
并维护其相应的 slot 信息。

#### 3.8 TaskManger
##### （1） 职责            
TaskManager 相当于整个集群的 Slave 节点，负责
- 具体的任务执行和
- 对应任务在每个节点上的资源申请和管理。
##### (2) 工作流程      
TaskManager 从 JobManager 接收需要部署的任务，然后使用 Slot 资源启动 Task，
建立数据接入的网络连接，接收数据并开始数据处理。
同时 TaskManager 之间的数据交互都是通过数据流的方式进行的。

##### 与MapReduce区别
可以看出，Flink 的任务运行其实是采用多线程的方式，    
这和 MapReduce 多 JVM 进行的方式有很大的区别，     
Flink 能够极大提高 CPU 使用效率，
在多个任务和 Task 之间通过 TaskSlot 方式共享系统资源，
每个 TaskManager 中通过管理多个 TaskSlot 资源池进行对资源进行有效管理



#### 3.9 Flink 计算资源的调度是如何实现的？
其实就是说Slot       
TaskManager中最细粒度的资源是Task slot，
代表了一个固定大小的资源子集，每个TaskManager会将其所占有的资源平分给它的slot。     

每个 TaskManager 有一个slot，也就意味着每个task运行在独立的 JVM 中。每个 TaskManager 有多个slot的话，也就是说多个task运行在同一个JVM中。   

而在同一个JVM进程中的task，
可以共享TCP连接（基于多路复用）和心跳消息，可以减少数据的网络传输，
也能共享一些数据结构，一定程度上减少了每个task的消耗

#### 3.10 Flink的数据抽象及数据交换过程？        
描述
- 内存抽象 MemorySegement  
MemorySegment就是Flink的内存抽象   32kb * 2048 64MB        
- 内存对象 Buffer   
Flink在数据从operator内的数据对象在向TaskManager上转移，
预备被发给下个节点的过程中，使用的抽象或者说内存对象是Buffer           
- 对象的转移的抽象是StreamRecord     
Java对象转为Buffer的中间对象的中间抽象是StreamRecord     

目的
- Flink 为了避免JVM的固有缺陷例如java对象存储密度低，
- FGC影响吞吐和响应等，实现了自主管理内存
    
  