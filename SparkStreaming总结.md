# 一、HelloWorld

## 1.SparkStreaming的原理

​		将数据流离散化为批，之后进行批处理。将流，按照设置的采集间隔，将每个采集周期获取到的数据，封装为一个RDD，之后对这个RDD再进行运算。

​		特点：  近实时(不是完全的实时)。 处理的单位(批次)。

​       优点：  在SparkStreaming模块调用Spark的其他API（core,sparksql,mlib,graphx）



## 2.SparkStreaming编程的套路

①创建入口 StreamingContext

②使用 StreamingContext 获取DStream(数据流)

③使用DStream的API，进行各种transformation

④将最终的DStream进行输出

⑤启动App

⑥阻塞当前App，让它一直运行，直到发命令让它停止



## 3.如何获取StreamingContext

```scala
//基于集群地址，app名称，和Duration(采集间隔，或周期)
def this(master: String,appName: String,batchDuration: Duration)

//基于SparkConf构造， 集群的url和app的名称都可以设置在SparkConf
 def this(conf: SparkConf, batchDuration: Duration)

//基于现有的SparkContext,将SparkConf构造为SparkContext
 def this(sparkContext: SparkContext, batchDuration: Duration)
```



## 4.如何构造Duration

```scala
//所有的case class都默认自带 apply()方法，当使用 类名()时默认调用类的apply()方法
Duration(n毫秒值)


//构造一个n毫秒的Duration
Milliseconds(n毫秒值)

//构造一个n秒的Duration
Seconds(n秒)

//构造一个n分钟的Duration
Minutes(n分钟)

```



## 5.如何从Kafka中获取一个DStream

①准备消费者的参数

```scala
val kafkaParams = Map[String, Object](
      "bootstrap.servers" -> "hadoop102:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "sz220409test",
      "auto.offset.reset" -> "latest",
      "enable.auto.commit" -> "true"
    )
```

②准备要消费的主题，一般情况下，一个流只消费一个主题

```scala
 val topics = Array("topicD")
```

③固定套路，从主题中，用消费者参数获取一个流

```scala
 val ds: InputDStream[ConsumerRecord[String, String]] = KafkaUtils.createDirectStream[String, String](
      streamingContext,
      PreferConsistent,
      Subscribe[String, String](topics, kafkaParams)
    )
```



注意事项：  获取的流中，存放的是ConsumerRecord[K,V],一般情况下，key中存放的是元数据，value中才是真正的数据。大部分情况，key不存任何信息，具体取决于生产者是如何生产的。在处理时，需要将 ConsumerRecord[K,V]的value取出，进行处理！

## 6.转换的注意事项

把DStream当做RDD来使用即可。

DStream就是无限的RDD集合，在每个Duration,DStream中获取到的数据都会自动封装为一个RDD，使用DStream进行的各种转换逻辑，都会套用到每一个RDD身上。



## 7.概念

抽象原语(简称原语):   DStream中的算子成为 原语，主要为了和RDD中同名的算子进行区别。



举例：

​			RDD.map:   map成为RDD中的算子

​			DStream.map:  map成为DStream中的抽象原语

​				

## 8.几个特殊的原语

### 8.1 Transform

tranform算子产生的背景：   DStream已经有大部分的和RDD中功能一样的算子，例如map,flatMap等，但是还缺少一部分算子，例如sortByKey等。 如果需要对DStream进行SortByKey操作时，它是无能为力的。

举例:

![image-20220823094935070](image-20220823094935070.png)

```scala
/*
	允许传入一个 transformFunc，它可以把 RDD[T] 处理为 RDD[U]，再把RDD[U]，封装为 DStream[U]返回。
*/
def transform[U: ClassTag](transformFunc: RDD[T] => RDD[U]): DStream[U]
```



**使用场景**:  当希望使用DStream没有，但是RDD中有的算子时，就使用Tranform，把对DStream的运算，改为对其中所包含的RDD的运算，再把结果包装为DStream返回。



### 8.2 foreachRDD

用于将数据写出到数据库。

```scala
// 调用foreachFunc，将RDD[T]输出，没有返回值
def foreachRDD(foreachFunc: RDD[T] => Unit): Unit
```

固定套路:

```scala
ds1.foreachRDD(rdd => {

      rdd.foreachPartition(partition => {

        //以分区为单位创建连接。 可以是对redis的连接，还可以是对mysql的连接等
        val jedis = new Jedis("hadoop102", 6379)

          //使用连接
        partition.foreach{
              //在当前word的基础上，再加上count，就是一个有状态的计算。借助redis提供的api实现了状态的累加
          case (word, count) => jedis.hincrBy("wordcount",word,count)
        }
        //关闭连接
        jedis.close()

      })

    })
```



## 9.有状态和无状态

状态(state)： 指当前计算中所需要存储的变量。

有状态的计算：  指当前的计算，需要之前计算中所存储的变量。

​								举例： 累加

​								在后续的练习中举例。

无状态的计算： 当前的计算，不需要之前计算中所存储的变量。

​								举例： 默认情况(hello)

​								目前讲的都是无状态的。

![image-20220823094914351](image-20220823094914351.png)

## 10.窗口操作

三个和时间相关的概念:

  batchDuration:     指没间隔多久采集到的数据作为一个批次(封装为一个RDD)。

  window           :     窗口(当前计算的数据所在的时间范围，当前要计算最近 多长时间的数据)。

​								窗口的方向只是是当前时间，往过去进行推移，不能包含未来时间。

 slide                 ： 滑动步长。指每间隔多久，触发一次计算(提交一个Job)。



举例： 将数据每间隔10s划分为1个批次，  要求每间隔1min ， 去计算过去30秒产生的数据。

```
batchDuration    ：   10s
window           ：   30s
slide            :    1min
```

​			

数量关系：  ①一般情况下， slide 和 window是相等的。

​				    如果slide  > window：  导致漏算(kafka除外)。

​					如果slide  < window :  导致重复算。

​					②slide 和 window必须是 batchDuration的整倍数！

​                     ③默认情况下，不指定window,和slide，window和slide都等于batchDuration



## 11.停止App

粗鲁地关闭:   直接kill掉程序即可。

优雅的关闭:    StreamingContext.stop()



关键问题：    StreamingContext.stop()写在哪里?

​						应该单独启动一个线程，这个线程和main线程是相互独立。在分线程中，不断监听是否需要关闭app，如果监听到了，就调用 StreamingContext.stop()。



# 二、从kafka获取DStream

## 1.从kafka获取DStream的功能

①kafka的分区和SparkRDD中的分区是1:1的关系。

​			如果希望从kafka获取的RDD有较大的初始分区数，应该怎么做?

​					增加kafka主题的分区。

②提供了访问offsets的功能



## 2.如何获取当前消费批次中数据的offsets(偏移量)

```scala
// OffsetRange就代表一个消费主题分区的偏移量信息，需要关注的是 OffsetRange.untilOffset,它代表你要提交的偏移量是多少
val ranges: Array[OffsetRange] = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
```

如果希望调用上述代码，必须在DStream的transform中或 foreachRDD中调用。

只有上述两个方法，将对DStream的运算转换为对RDD的运算



## 3.算子中代码运行的位置

![image-20220823153225624](image-20220823153225624.png)

## 4.获取offsets的注意事项

![image-20220823153813818](image-20220823153813818.png)



![image-20220823153952535](image-20220823153952535.png)

从kafka中获取一个DStream时，其实返回的是一个 DirectKafkaInputDStream。

而DirectKafkaInputDStream会周期性调用compute方法，将收到的数据封装为 KafkaRDD 



![image-20220823154105033](image-20220823154105033.png)



**结论**： 只有使用初始的DS，它才是DirectKafkaInputDStream，其中才有 KafkaRDD,才能获取偏移量。



## 5.提交Offsets到Kafka的_comsumeroffsets主题中

```scala
stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
```



![image-20220823161038050](image-20220823161038050.png)



**结论**:  只能用初始的DS，才是DirectKafkaInputDStream，才能调用 asInstanceOf[CanCommitOffsets].commitAsync(ranges)



## 6.提交offsets代码运行的位置

![image-20220823161725739](image-20220823161725739.png)

以上图是闭包产生的原因。

```
从原理上来说，偏移量在Driver端获取，一定在Driver端进行提交，提交的位置只能是RDD.算子的外面！
```