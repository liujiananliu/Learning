[TOC]



# Flink简介

# Flink部署



# Flink运行架构



# Flink流处理核心编程

## 基础部分

Flink的开发步骤，主要分为四大部分：

![image-20220905224639474](https://gitee.com/liujiananliu/upload-image/raw/master/202209052246502.png)

### Environment

### Source

### Transform

### Sink

## 高阶部分

### Window

#### 窗口概述

Window窗口是一种切割无限数据为有限块进行处理的手段

窗口把流切割成有限大小的多个”存储桶“(bucket)，在这些桶上进行计算

![image-20220905225620172](https://gitee.com/liujiananliu/upload-image/raw/master/202209052256197.png)

#### 窗口的分类

* 使用TimeWindow这个类来表示基于时间的窗口
  * 提供了key查询开始时间戳和结束时间戳的方法, 
  * 还提供了针对给定的窗口获取它允许的最大时间戳的方法maxTimestamp()

1. ##### 基于时间的窗口

   - ###### Tumbling Windows (滚动窗口)

      - 时间间隔可以通过`Time.milliseconds(x)` , `Time.seconds(x)`, ` Time.minutes(x)` 等来指定
      - 传递给window函数的对象叫**窗口分配器**

      ***sample code：***

      ```java
              env.socketTextStream("hadoop162", 9999)
                      .map(new MapFunction<String, WaterSensor>() {
                          @Override
                          public WaterSensor map(String value) throws Exception {
                              String[] data = value.split(",");
                              return new WaterSensor(
                                      data[0],
                                      Long.valueOf(data[1]),
                                      Integer.valueOf(data[2])
                              );
                          }
                      })
                      .assignTimestampsAndWatermarks(WatermarkStrategy.<WaterSensor>forBoundedOutOfOrderness(Duration.ofSeconds(3))
                              .withTimestampAssigner((element, recordTimestamp) -> element.getTs()))
                      .keyBy(WaterSensor::getId)
                      //定义一个长度为5s的窗口
                      .window(TumblingEventTimeWindows.of(Time.seconds(5)))
                      .process(new ProcessWindowFunction<WaterSensor, String, String, TimeWindow>() {
                          @Override
                          public void process(String s,
                                              ProcessWindowFunction<WaterSensor, String, String, TimeWindow>.Context context,
                                              Iterable<WaterSensor> elements,
                                              Collector<String> out) throws Exception {
                              ArrayList<WaterSensor> list = new ArrayList<>();
                              for (WaterSensor element : elements) {
                                  list.add(element);
                              }
                              String stt = AtguiguUtil.toDateTime(context.window().getStart());
                              String edt = AtguiguUtil.toDateTime(context.window().getEnd());
                              out.collect(s + " " + stt + " " + edt + " " + list);
                          }
                      })
                      .print();
      ```

      

   - ###### Sliding Windows (滑动窗口)

      ***sample code：***

      ```java
              env.socketTextStream("hadoop162", 9999)
                      .map(new MapFunction<String, WaterSensor>() {
                          @Override
                          public WaterSensor map(String value) throws Exception {
                              String[] data = value.split(",");
                              return new WaterSensor(
                                      data[0],
                                      Long.valueOf(data[1]),
                                      Integer.valueOf(data[2])
                              );
                          }
                      })
                      .keyBy(WaterSensor::getId)
                      .window(SlidingProcessingTimeWindows.of(Time.seconds(5), Time.seconds(3)))
                      .process(new ProcessWindowFunction<WaterSensor, String, String, TimeWindow>() {
                          @Override
                          public void process(String s,
                                              ProcessWindowFunction<WaterSensor, String, String, TimeWindow>.Context context,
                                              Iterable<WaterSensor> elements,
                                              Collector<String> out) throws Exception {
                              ArrayList<WaterSensor> list = new ArrayList<>();
                              for (WaterSensor element : elements) {
                                  list.add(element);
                              }
                              String stt = AtguiguUtil.toDateTime(context.window().getStart());
                              String edt = AtguiguUtil.toDateTime(context.window().getEnd());
                              out.collect(s + " " + stt + " " + edt + " " + list);
                          }
                      })
                      .print();
      ```

      

   - ###### Session Windows (会话窗口)

      因为会话窗口没有固定的开启和关闭时间, 所以会话窗口的创建和关闭与滚动,滑动窗口不同. 在Flink内部, 每到达一个新的元素都会创建一个新的会话窗口, 如果这些窗口彼此相距比较定义的gap小, 则会对他们进行合并. 为了能够合并, 会话窗口算子需要合并触发器和合并窗口函数: `ReduceFunction`, `AggregateFunction`, or `ProcessWindowFunction`

      1. 静态gap
         ```java
         .window(ProcessingTimeSessionWindows.withGap(Time.seconds(10)))
         ```

      2. 动态gap

         ```java
         .window(ProcessingTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor<Tuple2<String, Long>>() {
             @Override
             public long extract(Tuple2<String, Long> element) { // 返回 gap值, 单位毫秒
                 return element.f0.length() * 1000;
             }
         }))
         ```

      ***sample code：***

      ```java
      env.socketTextStream("hadoop162", 9999)
                      .map(new MapFunction<String, WaterSensor>() {
                          @Override
                          public WaterSensor map(String value) throws Exception {
                              String[] data = value.split(",");
                              return new WaterSensor(
                                      data[0],
                                      Long.valueOf(data[1]),
                                      Integer.valueOf(data[2])
                              );
                          }
                      })
                      .keyBy(WaterSensor::getId)
                      //定义一个长度为5s的窗口
                      .window(ProcessingTimeSessionWindows.withGap(Time.seconds(5)))
                      .process(new ProcessWindowFunction<WaterSensor, String, String, TimeWindow>() {
                          @Override
                          public void process(String s,
                                              ProcessWindowFunction<WaterSensor, String, String, TimeWindow>.Context context,
                                              Iterable<WaterSensor> elements,
                                              Collector<String> out) throws Exception {
                              ArrayList<WaterSensor> list = new ArrayList<>();
                              for (WaterSensor element : elements) {
                                  list.add(element);
                              }
                              String stt = AtguiguUtil.toDateTime(context.window().getStart());
                              String edt = AtguiguUtil.toDateTime(context.window().getEnd());
                              out.collect(s + " " + stt + " " + edt + " " + list);
                          }
                      })
                      .print();
      ```

      

2. ##### 全局窗口（Global Windows）(自定义触发器)

3. ##### 基于元素个数的窗口

   1. 滚动窗口
      ```java
      .countWindow(3) // 哪个窗口先达到3个元素, 哪个窗口就关闭. 不影响其他的窗口
      ```

   2. 滑动窗口
      ```java
      .countWindow(3, 2) // (window_size, sliding_size)
      ```

#### window Function

1. ReduceFunction (增量聚合函数)
2. AggregateFunction (增量聚合函数)
3. ProcessWindowFunction (全窗口函数)

#### Keyed vs Non-Keyed Windows

* 在keyed streams上使用窗口, 窗口计算被并行的运用在多个task上, 可以认为每个task都有自己单独窗口. 

* 在非non-keyed stream上使用窗口, 流的并行度只能是1, 所有的窗口逻辑只能在一个单独的task上执行.
  ```java
  .windowAll(TumblingProcessingTimeWindows.of(Time.seconds(10))) // 非key分区的流上使用window, 如果把并行度强行设置为>1, 则会抛出异常
  ```

  

### Watermark

#### 时间语义

1. **process time** 处理时间： 指的执行操作的各个设备的时间
   * **处理时间**是指的**执行操作的各个设备的时间**
   * 对于运行在处理时间上的流程序, 所有的基于时间的操作(比如时间窗口)都是使用的**设备时钟**.
2. **event time** 事件时间：指的这个事件发生的时间.
   * 事件时间是指的这个事件发生的时间
   * 在event进入Flink之前, 通常被嵌入到了event中, 一般作为这个event的时间戳存在.
   * 事件时间程序必须制定如何产生*Event Time Watermarks(水印)* . 在事件时间体系中, 水印是表示时间进度的标志(作用就相当于**现实时间的时钟**).

#### Watermark

- **测量事件时间的进度**的机制就是 **watermark(水印)**. 
- watermark作为数据流的一部分在流动, 并且携带一个时间戳t

- 有序流中的水印

  > watermark是流中一个简单的周期性的标记

  ![image-20220906001007822](https://gitee.com/liujiananliu/upload-image/raw/master/202209060010852.png)

- 乱序流中的水印

  > 水印是一种标记, 是流中的一个点, 所有在这个时间戳(水印中的时间戳)前的数据应该已经全部到达. 
  >
  > 一旦水印到达了算子, 则tu的值为这个水印的值.

  ![image-20220906001057042](https://gitee.com/liujiananliu/upload-image/raw/master/202209060010068.png)

#### 如何产生水印

> * 在 Flink 中， 水印由应用程序开发人员生成， 这通常需要对相应的领域有 一定的了解。完美的水印永远不会错：时间戳小于水印标记时间的事件不会再出现。在特殊情况下（例如非乱序事件流），最近一次事件的时间戳就可能是完美的水印。
>
> * 启发式水印则相反，它只估计时间，因此有可能出错， 即迟到的事件 （其时间戳小于水印标记时间）晚于水印出现。针对启发式水印， Flink 提供了处理迟到元素的机制。
>
> * 设定水印通常需要用到领域知识。举例来说，如果知道事件的迟到时间不会超过 5 秒， 就可以将水印标记时间设为收到的最大时间戳减去 5 秒。 另 一种做法是，采用一个 Flink 作业监控事件流，学习事件的迟到规律，并以此构建水印生成模型。

水印的计算 = 最大时间戳 - 最大乱序程度 - 1ms

水印的几个误区：

> 1. 水印是属于流，不是属于某个元素
> 2. 水印是作为特殊的数据插入到流中，随着数据的流动而流动
> 3. 事件时间窗口的关闭依据，就是看水印（定时器，触发器）

#### EventTime 和 WaterMark的使用

Flink内置了两个Watermark生成器

1. 时间戳单调增长
   ```java
   WatermarkStrategy.forMonotonousTimestamps();
   ```

2. 允许固定时间的延迟
   ```java
   WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10));
   ```

#### 自定义WatermarkStrategy

两种风格的Watermark生产方式：periodic(周期性), punctuated(打点式/间歇性)，都需要继承接口：WatermarkGenerator

1. periodic (周期性)
   ```java
   .assignTimestampsAndWatermarks(new WatermarkStrategy<WaterSensor>() {
       @Override
       public WatermarkGenerator<WaterSensor> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
           return new WatermarkGenerator<WaterSensor>() {
               Long maxTs = Long.MIN_VALUE + 3000 + 1;
               @Override
               public void onEvent(WaterSensor event, long eventTimestamp, WatermarkOutput output) {
                   System.out.println("Flink02_Window_WMK_Custom.onEvent");
                   maxTs = Math.max(maxTs, eventTimestamp);
                   // output.emitWatermark(new Watermark(maxTs));
               }
   
               @Override
               public void onPeriodicEmit(WatermarkOutput output) {
                   System.out.println("Flink02_Window_WMK_Custom.onPeriodicEmit");
                   output.emitWatermark(new Watermark(maxTs));
               }
           };
       }
   }.withTimestampAssigner(new SerializableTimestampAssigner<WaterSensor>() {
       @Override
       public long extractTimestamp(WaterSensor element, long recordTimestamp) {
           return element.getTs();
       }
   }))
   ```

   

2. punctuated (打点式/间歇性)
   ```java
   .assignTimestampsAndWatermarks(new WatermarkStrategy<WaterSensor>() {
       @Override
       public WatermarkGenerator<WaterSensor> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context) {
           return new WatermarkGenerator<WaterSensor>() {
               Long maxTs = Long.MIN_VALUE + 3000 + 1;
               @Override
               public void onEvent(WaterSensor event, long eventTimestamp, WatermarkOutput output) {
                   System.out.println("Flink02_Window_WMK_Custom.onEvent");
                   maxTs = Math.max(maxTs, eventTimestamp);
                   output.emitWatermark(new Watermark(maxTs));
               }
   
               @Override
               public void onPeriodicEmit(WatermarkOutput output) {
                   System.out.println("Flink02_Window_WMK_Custom.onPeriodicEmit");
                   // output.emitWatermark(new Watermark(maxTs));
               }
           };
       }
   }.withTimestampAssigner(new SerializableTimestampAssigner<WaterSensor>() {
       @Override
       public long extractTimestamp(WaterSensor element, long recordTimestamp) {
           return element.getTs();
       }
   }))
   ```

#### 多并行度下的Watermark传递

生成水印：每个并行度根据当前并行度内最大时间戳计算得到

水印的传递：广播传递：从收到的所有水印中找一个最新的传递到后面的所有并行度，找到最小的水印进行传递，木桶原理

#### 数据倾斜导致的水印不更新问题

> 1. 解决数据倾斜
>
> 2. source的并行度改成1（不建议）
>
> 3. map之前平均分布一下数据rebalence,shuffle（可以考虑）
>
> 4. 把不更新数据的并行度标记为空闲，水印传递就以其他为准
>    ```java
>    .assignTimestampsAndWatermarks(WatermarkStrategy.<WaterSensor>forBoundedOutOfOrderness(Duration.ofSeconds(3))
>                                   .withTimestampAssigner((element, recordTimestamp) -> element.getTs())
>                                   //如果一个并行度10s没有更新(没有更新水印),水印传递就以其他的为准
>                                   .withIdleness(Duration.ofSeconds(10)))
>    ```
>
>    

#### 窗口允许迟到的数据

当触发了窗口计算后，会先计算当前的结果，但是此时并不会关闭窗口，以后来一条吃到数据，触发一次这条数据所在窗口计算（增量计算）

什么时候真正关闭窗口？watermark 超过了窗口结束时间 + 等待时间

```java
.window(TumblingEventTimeWindows.of(Time.seconds(5)))
.allowedLateness(Time.seconds(2)) //等待时间
```

process：数据重复计算的问题

增加聚合：没有重复计算的问题

> 允许迟到只能运用在event time 上

### SideOutPut

#### 接收窗口关闭之后的迟到数据

窗口真正关闭之后，这时候来的属于关闭窗口的数据，会直接进入侧输出流

```java
.window(TumblingEventTimeWindows.of(Time.seconds(5)))
//把真正迟到的数据写入到侧输出流中
//做成匿名内部类的方式，来保存泛型到运行时
.sideOutputLateData(new OutputTag<WaterSensor>("late"){})
...
normal.print("正常数据");
normal.getSideOutput(new OutputTag<WaterSensor>("late"){}).print("迟到数据");
```

***注意：***

```java
//做成匿名内部类的方式，来保存泛型到运行时
.sideOutputLateData(new OutputTag<WaterSensor>("late"){})
```



#### 使用侧输出流把一个流拆成多个流

```java
//只有在process算子里才能实现测输出流的分流
.process(new ProcessFunction<WaterSensor, String>() {
    @Override
    public void processElement(WaterSensor value,
                               ProcessFunction<WaterSensor, String>.Context ctx,
                               Collector<String> out) throws Exception {
        if ("s1".equals(value.getId())) {
            out.collect(value.toString());
        } else if ("s2".equals(value.getId())) {
            ctx.output(new OutputTag<String>("侧2"){}, JSON.toJSONString(value));
        } else {
            ctx.output(new OutputTag<WaterSensor>("other"){}, value);
        }
    }
});
main.print("主流");
main.getSideOutput(new OutputTag<String>("侧2"){}).print("测输出流2");
main.getSideOutput(new OutputTag<WaterSensor>("other"){}).print("其他");
```

***注意：***

```java
//只有在process算子里才能实现测输出流的分流
```



### ProcessFunction API



### 定时器

基于处理时间的定时器

基于事件时间的定时器





### 状态编程

#### 状态的分类

#### 算子状态的使用

#### 监控状态的使用

#### 状态后端









# Flink CEP编程

# Flink SQL编程

