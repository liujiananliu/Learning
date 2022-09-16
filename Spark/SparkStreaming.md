

# 一、SparkStreaming编程基础



## 1.SparkStreaming原理

将数据流离散化为批，之后进行批处理。将流，按照设置的采集间隔，将每个采集周期获取到的数据，封装为一个RDD，之后对这个RDD再进行运算。

* 特点：近实时(不是完全的实时)。 处理的单位(批次)。
* 在SparkStreaming模块调用Spark的其他API（core,sparksql,mlib,graphx）

## 2.SparkStreaming编程的套路

==编程套路：== 

> 1. 创建入口StreamingContext
> 2. 使用StreamingContext获取DStream(数据流)
> 3. 使用DStream的API，进行各种transformation
> 4. 将最终用的DStreaming进行输出
> 5. 启动App
> 6. 阻塞当前App，让他一直运行，直到发命令让他停止

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

1. 准备消费者的参数
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

   

2. 准备要消费的主题，一般情况下，一个流只消费一个主题
   ```scala
    val topics = Array("topicD")
   ```

3. 固定套路，从主题中，用消费者参数获取一个流
   ```scala
    val ds: InputDStream[ConsumerRecord[String, String]] = KafkaUtils.createDirectStream[String, String](
         streamingContext,
         PreferConsistent,
         Subscribe[String, String](topics, kafkaParams)
       )
   ```

   *注意事项*： 获取的流中，存放的是ConsumerRecord[K,V],一般情况下，key中存放的是元数据，value中才是真正的数据。大部分情况，key不存任何信息，具体取决于生产者是如何生产的。在处理时，需要将 ConsumerRecord[K,V]的value取出，进行处理！

## 6.转换的注意事项

把DStream当做RDD来使用即可。

DStream就是无限的RDD集合，在每个Duration,DStream中获取到的数据都会自动封装为一个RDD，使用DStream进行的各种转换逻辑，都会套用到每一个RDD身上。



## 7.原语

> 抽象原语(简称原语): DStream中的算子成为 原语，主要为了和RDD中同名的算子进行区别。
>
> 举例：
>
>  RDD.map: map成为RDD中的算子
>
>  DStream.map: map成为DStream中的抽象原语

几个特殊的原语：

### 7.1 Transform

tranform算子产生的背景： DStream已经有大部分的和RDD中功能一样的算子，例如map,flatMap等，但是还缺少一部分算子，例如sortByKey等。 如果需要对DStream进行SortByKey操作时，它是无能为力的。

举例:	ds1.sortByKey

```scala
/*
	允许传入一个 transformFunc，它可以把 RDD[T] 处理为 RDD[U]，再把RDD[U]，封装为 DStream[U]返回。
*/
def transform[U: ClassTag](transformFunc: RDD[T] => RDD[U]): DStream[U]
```

**使用场景**: 当希望使用DStream没有，但是RDD中有的算子时，就使用Tranform，把对DStream的运算，改为对其中所包含的RDD的运算，再把结果包装为DStream返回。

### 7.2 foreachRDD

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

## 8.设计redis中的key-value

***参考粒度***

1. 所有的单词是一个k-v
   ```
   key: wordcount
   value: hash(map)
   ```

2. 一个单词一个k-v
   ```
   key: word(单词)
   value: 个数
   ```

***==只要是获取数据库连接，以分区为单位获取。 一个分区共享一个连接，而不是分区中的每个数据各自用各自的。==***

## 9.窗口操作

三个和时间相关的概念:

**batchDuration:** 指没间隔多久采集到的数据作为一个批次(封装为一个RDD)。

**window :** 窗口(当前计算的数据所在的时间范围，当前要计算最近 多长时间的数据)。
				 窗口的方向只是是当前时间，往过去进行推移，不能包含未来时间。

**slide** ： 滑动步长。指每间隔多久，触发一次计算(提交一个Job)。

举例： 将数据每间隔10s划分为1个批次， 要求每间隔1min ， 去计算过去30秒产生的数据。

```
batchDuration    ：   10s
window           ：   30s
slide            :    1min
```

数量关系：

1. 一般情况下， slide 和 window是相等的。
    如果slide > window： 导致漏算(kafka除外)。
    如果slide < window : 导致重复算。

2. slide 和 window必须是 batchDuration的整倍数！

3. 默认情况下，不指定window,和slide，window和slide都等于batchDuration

## 10.停止App

**粗鲁地关闭:** 直接kill掉程序即可。

**优雅的关闭:** StreamingContext.stop()

​	 关键问题： StreamingContext.stop()写在哪里?

​	 应该单独启动一个线程，这个线程和main线程是相互独立。在分线程中，不断监听是否需要关闭app，如果监听到了，就调用		StreamingContext.stop()。

```scala
object GracefullyStopAppDemo {
  def main(args: Array[String]): Unit = {
    val streamingContext = new StreamingContext("local", "testAppName1",Seconds(5))
      
    val kafkaParams = Map[String, Object](
      "bootstrap.servers" -> "hadoop102:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "testGroupId",
      "auto.offset.reset" -> "latest",
      "enable.auto.commit" -> "true"
    )
      
    val topics = Array("topicB")
      
    val ds = KafkaUtils.createDirectStream[String, String](
      streamingContext,
      PreferConsistent,
      Subscribe[String, String](topics, kafkaParams)
    )
      
    val ds1: DStream[(String, Int)] = ds.flatMap(record => record.value().split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
    ds1.print(1000)
    streamingContext.start()
    val stopAppTread = new StopAppTread("stop", streamingContext)
    new Thread(stopAppTread).start()
    streamingContext.awaitTermination()
  }
}

class StopAppTread(name: String, context: StreamingContext) extends Runnable {
  private val jedis = new Jedis("hadoop103", 6379)
  override def run(): Unit = {
    if(!jedis.exists(name)){
      Thread.sleep(5000)
    }
    context.stop(true, true)
  }
}
```



# 二、从kafka中获取和写入offsets

> [Spark Streaming + Kafka Integration Guide](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)

1. kafka的分区和SparkRDD中的分区是1:1的关系。
   如果希望从kafka获取的RDD有较大的初始分区数，应该怎么做?
   增加kafka主题的分区。

2. 提供了访问offsets的功能

   - **Obtaining Offsets**

     ```scala
     stream.foreachRDD { rdd =>
       val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
       rdd.foreachPartition { iter =>
         val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
         println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
       }
     }
     ```

   - **Storing Offsets**

     ```scala
     stream.foreachRDD { rdd =>
       val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
     
       // some time later, after outputs have completed
       stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
     }
     ```

     ```
     注1：
     OffsetRange： 代表主题中一个分区的offset信息
     所消费的主题有几个分区，就会有几个OffsetRange，把这些OffsetRange封装为集合  Array[OffsetRange]
     
     注2：
     HasOffsetRanges 只有一个子类:  KafkaRDD
     只有当rdd是 KafkaRDD 类型时，此行代码才不会报错！
     结论： 只有初始DS，其中封装的RDD才是kafkaRDD
     ```

3. 提交offsets到kafka

   1. 自动提交

      在消费者的参数上设置    "enable.auto.commit" -> "true"

   2. 手动提交

      ```
      1.取消自动提交 "enable.auto.commit" -> "false"
      2.获取到当前批次的偏移量，尤其关注 offsetRange中的utilOffset'
      3.调用API，把每个分区消费到的utilOffset提交到kafka集群的_comsumer_offsets中
      ```









# 三、Exactly Once 消费

## 支持输出幂等性的数据库

- **HBase**
  hbase借助rowkey，保证输出的幂等性

  > `put (r1,cf1:name=jack)`
  > `put (r1,cf1:name=jack)`	$\Longrightarrow$	 `(r1,cf1:name=jack)`

- **Redis**
  redis只要key相同，也有幂等性

  > `set (k1,v1)`
  > `set (k1,v1)`	$\Longrightarrow$	 `set (k1,v1)`

- **ES**

  es的put有幂等性

  > `PUT /库名/表名/id=1, {数据}`
  > `PUT /库名/表名/id=1, {数据}`	$\Longrightarrow$	 `id= 1,id=1,{数据}`

- **Mysql**

  有两种插入数据的方式具有幂等性
  
  > `insert into ...` 不行，主键冲突
  > `insert into ... on duplicate key update...` --> 可实现幂等性
  > `replace into ...` --> *可实现幂等性*



## exactly once 实现

### 方式一

**at least once + 幂等性输出(去重) = exactly once**

==编程套路：== 

> 1. 取消offsets的自动提交
> 2. 获取偏移量
> 3. 幂等输出
> 4. 在输出后手动提交

```scala
  def main(args: Array[String]): Unit = {
    //创建StreamingContext
    val streamingContext = new StreamingContext("local[*]", "testAppName", Duration(5000))

    //使用StreamingContext获取DStream(数据流)
    val kafkaParams = Map[String, Object](
      "bootstrap.servers" -> "hadoop102:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "testGroupId",
      "auto.offset.reset" -> "latest",
      //①取消offsets的自动提交
      "enable.auto.commit" -> "false"
    )
    val topics = Array("topicA")
    val ds: InputDStream[ConsumerRecord[String, String]] = KafkaUtils.createDirectStream[String, String](
      streamingContext,
      PreferConsistent,
      Subscribe[String, String](topics, kafkaParams)
    )

    ds.foreachRDD(rdd => {
      //②获取偏移量
      val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

      //③幂等输出
      rdd.foreach(record => {
        //幂等输出：这里是伪代码，只是介绍此种编程的基本套路，具体输出什么，要看业务！
      })

      //④在输出后手动提交offsets
      ds.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
    })
    streamingContext.start()
    streamingContext.awaitTermination()
  }
```



### 方式二

**at least once + 将结果和offsets在一个事务中写出 = exactly once**

[^注意：]:***关系型数据库才有事务*，Mysql中只有Innodb引擎，才有事务**

==编程套路：== 

> 1. 程序启动前，要从关系型数据库读取 上次提交的 offsets
>    案例中写了一个readHistoryOffsetsFromMysql方法
> 2. 基于上次提交的offsets获取一个，从提交位置向后消费的 DStream
> 3. 获取当前批次数据的offsets
> 4. 进行业务的计算
> 5. 将分布式计算的结果收集到Driver端，和offsets在一个事务中写出
>    案例中写了一个writeDataAndOffsetsInATransction方法

```scala
object TransactionExactlyOnce {
  val groupId = "testGroupId"
  val topic = "topicB"

  def readHistoryOffsetsFromMysql(groupId: String, topic: String): mutable.Map[TopicPartition, Long] = {
    val offsets = new mutable.HashMap[TopicPartition, Long]()
    var connection: Connection = null
    var ps: PreparedStatement = null

    //②准备sql，使用Connection预编译sql，获取PreparedStatement
    val sql: String =
      """
        |select
        |   partitionId,untilOffset
        |from offsets
        |where groupId = ? and topic = ?
        |""".stripMargin

    try {
      //①准备Connection
      connection = JDBCUtil.getConnection()
      ps = connection.prepareStatement(sql)
      ps.setString(1, groupId)
      ps.setString(2, topic)
      val resultSet: ResultSet = ps.executeQuery()
      while (resultSet.next()) {
        offsets.put(new TopicPartition(topic, resultSet.getInt("partitionId")),
          resultSet.getLong("untilOffset"))
      }
    } catch {
      case e: Exception => e.printStackTrace()
    } finally {
      if (ps != null) ps.close()
      if (connection != null) connection.close()
    }
    offsets
  }
  
  def writeDataAndOffsetsInTransaction(data: Array[(String, Int)], offsetRanges: Array[OffsetRange]) = {
    var connection: Connection = null
    var ps1: PreparedStatement = null
    var ps2: PreparedStatement = null
    val sql1: String =
      """
        |INSERT INTO `wordcount` VALUES(?,?)
        |ON DUPLICATE KEY UPDATE COUNT = COUNT + VALUES(COUNT)
        |""".stripMargin
    val sql2: String =
      """
        |REPLACE INTO `offsets` VALUES(?,?,?,?)
        |""".stripMargin

    try {

      //①获取Connection
      connection = JDBCUtil.getConnection()

      //体现事务：取消事务的自动提交
      connection.setAutoCommit(false)

      //②准备sql，使用connection预编译sql，获取PreparedStatement
      ps1 = connection.prepareStatement(sql1)
      ps2 = connection.prepareStatement(sql2)

      //③向占位符中添加参数
      for ((word, count) <- data) {
        ps1.setString(1, word)
        ps1.setInt(2, count)
        ps1.addBatch()
      }
      for (offsetRange <- offsetRanges) {
        ps2.setString(1, groupId)
        ps2.setString(2, topic)
        ps2.setInt(3, offsetRange.partition)
        ps2.setLong(4, offsetRange.untilOffset)
        ps2.addBatch()
      }

      //④执行sql
      val writeDataSuccess: Array[Int] = ps1.executeBatch()
      val writeOffsetsSuccess: Array[Int] = ps2.executeBatch()

      //体现事务，提交事务
      connection.commit()

      //⑤输出返回值
      println("数据成功写入了：" + writeDataSuccess.size)
      println("offsets成功写入了：" + writeOffsetsSuccess.size)

    } catch {
      case e: Exception =>
        //体现事务，回滚
        connection.rollback()
        e.printStackTrace()
    } finally {
      if (ps1 != null) ps1.close()
      if (ps2 != null) ps2.close()
      if (connection != null) connection.close()
    }
  }

  def main(args: Array[String]): Unit = {
    //①程序启动前，要从关系型数据库读取上次提交的offsets
    val offsets: mutable.Map[TopicPartition, Long] = readHistoryOffsetsFromMysql(groupId: String, topic: String)
    val streamingContext = new StreamingContext("local[*]", "testAppName1", Seconds(5))

    val kafkaParams = Map[String, Object](
      "bootstrap.servers" -> "hadoop102:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> groupId,
      "auto.offset.reset" -> "latest",
      "enable.auto.commit" -> "false"
    )

    //②基于上次提交的offsets获取一个，从提交位置向后消费的DStream
    val topics = Array(topic)
    val ds = KafkaUtils.createDirectStream[String, String](
      streamingContext,
      PreferConsistent,
      Subscribe[String, String](topics, kafkaParams, offsets)
    )
    ds.foreachRDD(rdd => {
      if (!rdd.isEmpty()) { //③获取当前批次数据的offsets
        val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

        //④进行业务的计算
        val rdd1: RDD[(String, Int)] = rdd.flatMap(record => record.value().split(" ")).map(word => (word, 1)).reduceByKey(_ + _)

        //⑤将分布式计算的结果收集到Driver端，和offsets在一个事务中写出
        val data: Array[(String, Int)] = rdd1.collect()
        writeDataAndOffsetsInTransaction(data, offsetRanges)
        //同时把offsets也提交到kafka.没用，下次读取从mysql读
        ds.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
      }
    })

    streamingContext.start()
    streamingContext.awaitTermination()
  }
}
```



# 四、Spark Streaming example

```scala
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

// scalastyle:off println
package org.apache.spark.examples.streaming

import org.apache.spark.SparkConf
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.{Seconds, StreamingContext}

/**
 * Counts words in UTF8 encoded, '\n' delimited text received from the network every second.
 *
 * Usage: NetworkWordCount <hostname> <port>
 * <hostname> and <port> describe the TCP server that Spark Streaming would connect to receive data.
 *
 * To run this on your local machine, you need to first run a Netcat server
 *    `$ nc -lk 9999`
 * and then run the example
 *    `$ bin/run-example org.apache.spark.examples.streaming.NetworkWordCount localhost 9999`
 */
object NetworkWordCount {
  def main(args: Array[String]): Unit = {
    if (args.length < 2) {
      System.err.println("Usage: NetworkWordCount <hostname> <port>")
      System.exit(1)
    }

    StreamingExamples.setStreamingLogLevels()

    // Create the context with a 1 second batch size
    val sparkConf = new SparkConf().setAppName("NetworkWordCount")
    val ssc = new StreamingContext(sparkConf, Seconds(1))

    // Create a socket stream on target ip:port and count the
    // words in input stream of \n delimited text (e.g. generated by 'nc')
    // Note that no duplication in storage level only for running locally.
    // Replication necessary in distributed scenario for fault tolerance.
    val lines = ssc.socketTextStream(args(0), args(1).toInt, StorageLevel.MEMORY_AND_DISK_SER)
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
// scalastyle:on println
```

[^官方案例源码链接]: [NetworkWordCount](https://github.com/apache/spark/blob/v3.3.0/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala)
