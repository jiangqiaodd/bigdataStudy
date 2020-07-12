### window和watermark原理
![window和watermark原理](/src/resource/windowWatermarkSrc.png)

#### 1.1 watermark的生成
- Flink通过水印分配器（TimestampsAndPeriodicWatermarksOperator和TimestampsAndPunctuatedWatermarksOperator这两个算子）向事件流中注入水印。  
- 元素在streaming dataflow引擎中流动到WindowOperator时，会被分为两拨，分别是普通事件和水印。 

#### 1.2 window的组成和处理逻辑
##### （1） window 三组件
- windowAssigner        
负责将元素分配到不同的window。
- trigger   
Trigger即触发器，定义何时或什么情况下Fire一个window
- evictor(可选    
驱逐者，即保留上一window留下的某些元素。 
- 内置对象Timer


##### （2） Flink 的窗口机制以及各组件之间是如何相互工作：
- assign分配到不同window     
数据流源源不断地进入算子（window operator），每一个到达的元素都会被交给 WindowAssigner。WindowAssigner 会决定元素被放到哪个或哪些窗口（window），可能会创建新窗口。因为一个元素可以被放入多个窗口中（个人理解是滑动窗口，滚动窗口不会有此现象），所以同时存在多个窗口是可能的。注意，    
Window本身只是一个ID标识符，其内部可能存储了一些元数据，如TimeWindow中有开始和结束时间，但是并不会存储窗口中的元素。窗口中的元素实际存储在 Key/Value State 中，key为Window，value为元素集合（或聚合值）。为了保证窗口的容错性，该实现依赖了 Flink 的 State 机制。
- 每一个窗口都拥有一个属于自己的 Trigger，Trigger上会有定时器，用来决定一个窗口何时能够被计算或清除。     
每当有元素加入到该窗口，或者之前注册的定时器超时了，那么Trigger都会被调用。       
Trigger的返回结果可以是 continue（不做任何操作），fire（处理窗口数据），purge（移除窗口和窗口中的数据），或者 fire + purge。       
一个Trigger的调用结果只是fire的话，那么会计算窗口并保留窗口原样，也就是说窗口中的数据仍然保留不变，等待下次Trigger fire的时候再次执行计算。一个窗口可以被重复计算多次直到它被 purge 了。在purge之前，窗口会一直占用着内存。     
- 当Trigger fire了，窗口中的元素集合就会交给Evictor（如果指定了的话）。      
Evictor 主要用来遍历窗口中的元素列表，并决定最先进入窗口的多少个元素需要被移除。剩余的元素会交给用户指定的函数进行窗口的计算。如果没有 Evictor 的话，窗口中的所有元素会一起交给函数进行计算。
- 计算函数收到了窗口的元素（可能经过了 Evictor 的过滤）       
并计算出窗口的结果值，并发送给下游。窗口的结果值可以是一个也可以是多个。DataStream API 上可以接收不同类型的计算函数，包括预定义的sum(),min(),max()，还有 ReduceFunction，FoldFunction，还有WindowFunction。WindowFunction 是最通用的计算函数，其他的预定义的函数基本都是基于该函数实现的。
- Flink 对于一些聚合类的窗口计算（如sum,min）做了优化，
因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个result值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改result值。
这样可以大大降低内存的消耗并提升性能。但是如果用户定义了 Evictor，则不会启用对聚合窗口的优化，因为 Evictor 需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

#### 1.3 Flink中 window Function的使用    
##### （1） 分组
分组的stream调用keyBy(…)和window(…)       
stream.keyBy().window().trigger().evictor().allowlateness().reduce/apply
        
非分组的stream中window()换成了windowAll(…)      
stream.windowAll().trigger().evictor().allowlateness().reduce/apply

##### （2) Trigger       
 每个 Window 都有一个Trigger，Trigger(触发器)指定了函数在什么条件下可被应用(函数何时被触发),
 一个触发策略可以是 "当窗口中的元素个数超过9个时" 或者 "当水印达到窗口的边界时"。触发器还可以决定在窗口创建和删除之间的任意时刻清除窗口的内容
 如果默认的触发器不能满足你的需要，你可以通过调用trigger(...)来指定一个自定义的触发器。触发器的接口有5个方法来允许触发器处理不同的事件:
- onElement()方法,每个元素被添加到窗口时调用       
返回值：TriggerResult.CONTINUE/Fire/Purge
- onEventTime()方法,当一个已注册的事件时间计时器启动时调用
- onProcessingTime()方法,当一个已注册的处理时间计时器启动时调用
- onMerge()方法，与状态性触发器相关，当使用会话窗口时，两个触发器对应的窗口合并时，合并两个触发器的状态。
- clear()方法执行任何需要清除的相应窗口
 ```
onEventTime && onProcessingTime 根据定时器触发，从而判定处理窗口的逻辑
如果是水印（事件时间场景），则方法processWatermark将会被调用，它将会处理水印定时器队列中的定时器。如果时间戳满足条件，则利用触发器的onEventTime方法进行处理。
而对于处理时间的场景，WindowOperator将自身实现为一个基于处理时间的触发器，以触发trigger方法来消费处理时间定时器队列中的定时器满足条件则会调用窗口触发器的onProcessingTime，根据触发结果判断是否对窗口进行计算
```
 ##### (3) 内置的触发器
 Flink有一些内置的触发器:
- EventTimeTrigger(前面提到过)触发是根据由水印衡量的事件时间的进度来的
- ProcessingTimeTrigger 根据处理时间来触发
- CountTrigger 一旦窗口中的元素个数超出了给定的限制就会触发
- PurgingTrigger 作为另一个触发器的参数并将它转换成一个清除类型

##### （4） Timer     
WindowOperator的内部类Timer。Timer是所有时间窗口执行的基础，它其实是一个上下文对象，封装了三个属性：  
timestamp：触发器触发的时间戳；    
key：当前元素所归属的分组的键；   
window：当前元素所属窗口； 

在注册定时器时，会新建定时器对象并将其加入到定时器队列中。   
等到时间相关的处理方法（processWatermark和trigger）被触发调用，
则会从定时器队列中消费定时器对象并调用窗口触发器的onProcessTime onEventTime方法
根据返回结果是Fire/Continue等来判断是否触发窗口的计算   
#### 1.3 窗口函数 window function
##### (1) 增量聚合
指窗口每进入一条数据就计算一次     
![增量聚合](/src/resource/addedCaclulate.png)  
- reduce(reduceFunction)  
- aggregation(aggregationFunction) 
- sum()
- min()
- max()  
##### (2) 全量聚合
指在窗口触发的时候才会对窗口内的所有数据进行一次计算（等窗口的数据到齐，才开始进行聚合计算，可实现对窗口内的数据进行排序等需求）
![增量聚合](/src/resource/allCacalulate.png)
- apply(windowFunction)
- process(processWindowFunction)
对应的函数传入的是一个 Iteration