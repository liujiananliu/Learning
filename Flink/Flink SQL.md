[TOC]



# 核心概念

Flink有两种关系型API来做流批统一处理：Table API 和 SQL。

这两种API中的查询对于批(DataSet)和流(DataStream)的输入有相同的语义，也会产生同样的计算结果。

## 动态表（Dynamic Tables）和连续查询（Continuous Query）

![image-20220915203933510](https://gitee.com/liujiananliu/upload-image/raw/master/202209152039571.png)

1. 将流转换为动态表
2. 在动态表上计算一个连续查询，生成一个新的动态表
3. 生成的动态表被转换回流



# Flink Table API

## 基本使用

表与DataStream的混合使用：

```java
// 1. 创建表的执行环境
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
// 2. 创建表: 将流转换成动态表. 表的字段名从pojo的属性名自动抽取
Table table = tEnv.fromDataStream(stream);
// 3. 对动态表进行查询
Table result = table
    .where($("id").isEqual("sensor_1"))
    .select($("id"), $("ts"), $("vc"));
// 4. 把动态表转换成流
DataStream<Row> resultStream = tEnv.toAppendStream(resultTable, Row.class);
resultStream.print();
```

聚合操作：

```java
// 3. 对动态表进行查询
Table result = table
    .where($("vc").isGreaterOrEqual(20))
    .groupBy($("id"))
    .aggregate($("vc").sum().as("vc_sum"))
    .select($("id"), $("vc_sum"));
// 4. 把动态表转换成流 如果涉及到数据的更新, 要用到撤回流. 多个了一个boolean标记
DataStream<Tuple2<Boolean, Row>> resultStream = tableEnv.toRetractStream(result, Row.class);
//打印最新的数据
resultStream
    .filter(t -> t.f0) // 过滤掉撤回数据
    .map(t -> t.f1) // 只要row
    .print();
```

## 表到流的转换：

### Append-only流

### Retract流

### Upsert流

## 通过Connector声明写入/写出数据

前面是先得到流，再转成动态表，其实动态表也可以直接连接/写出到数据

使用Connector的方法已经过时，推荐使用sql实现动态表直接和文件关联

# Flink SQL

## 查询未注册的表

```java
Table inputTable = tEnv.fromDataStream(waterSensorStream);
Table table = tEnv.sqlQuery("select * from " + inputTable + " where id='sensor_1'");
```

## 查询已注册的表

```java
Table inputTable = tableEnv.fromDataStream(waterSensorStream);
tEnv.createTemporaryView("sensor", inputTable);
Table table = tEnv.sqlQuery("select * from sensor where id='sensor_1'");
```

## 通过DDL方式建表，直接和文件关联

将来从这个表读取数据，就自动从文件读取 (官方推荐使用这种)

```java
tEnv.executeSql("create table sensor(" +
                " id string, " +
                " ts bigint, " +
                " vc int" +
                ")with(" +
                " 'connector' = 'filesystem', " +
                " 'path' = 'input/sensor.txt', " +
                " 'format' = 'csv' " +
                ")");
Table result = tEnv.sqlQuery("select * from sensor where id='sensor_1'");
tEnv.executeSql("create table sensor_out(" +
                " id string, " +
                " ts bigint, " +
                " vc int" +
                ")with(" +
                " 'connector' = 'filesystem', " +
                " 'path' = 'input/test', " +
                " 'format' = 'json' " +
                ")");

result.executeInsert("sensor_out");
```

## kafka到kafka

```java
//从kafka读数据
tEnv.executeSql("create table sensor(" +
                " id string, " +
                " ts bigint, " +
                " vc int" +
                ")with(" +
                "  'connector' = 'kafka', " +
                "  'topic' = 's1', " +
                "  'properties.bootstrap.servers' = 'hadoop162:9092', " +
                "  'properties.group.id' = 'atguigu', " +
                "  'scan.startup.mode' = 'latest-offset', " +
                "  'format' = 'csv'" +
                ")");

//写入到kafka
tEnv.executeSql("create table sensor_out(" +
                " id string, " +
                " ts bigint, " +
                " vc int" +
                ")with(" +
                "  'connector' = 'kafka', " +
                "  'topic' = 's2', " +
                "  'properties.bootstrap.servers' = 'hadoop162:9092', " +
                "  'format' = 'json'" +
                ")");

Table result = tEnv.sqlQuery("select id, ts, vc from sensor");
result.executeInsert("sensor_out");
```

## Upsert Kafka

> [Upsert Kafka | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/connectors/table/upsert-kafka/)

普通的kafka不能写入update change数据，只能写入append only数据

如果确实有需求要写入变化数据, 怎么办? **使用upsert kafka**

*注意：确保在 DDL 中定义主键。*

```java
tEnv.executeSql("create table sensor(" +
                " id string, " +
                " vc int ," +
                " primary key (`id`)NOT ENFORCED " + // flink不检测主键的特性: 非空不重复, flink检测不了, 默认是强迫检查
                ")with(" +
                "  'connector' = 'upsert-kafka', " +
                "  'topic' = 's3', " +
                "  'properties.bootstrap.servers' = 'hadoop162:9092', " +
                "  'key.format' = 'json', " +
                "  'value.format' = 'json'" +
                ")");
```

如果数据有更新，写入到kafka的时候，必须使用upsert-kafka
如果从upsert-kafka写入的topic中读取的时候，可以使用upsert-kafka，也可以使用普通的kafka，一般使用普通的kafka



# 时间属性

## 处理时间：

处理时间属性可以在 schema 定义的时候用 `.proctime` 后缀来定义。时间属性一定**不能定义在一个已有字段上**，所以它**新增一个字段_**

### Table API：

```java
Table table = tEnv.fromDataStream(stream, $("id"), $("ts"), $("vc"), $("pt").proctime());
table.printSchema();
table.execute().print();
```

### SQL：

```java
tEnv.executeSql("create table sensor(" +
                "   id string, " +
                "   ts bigint, " +
                "   vc int, " +
                "   pt as proctime()" +
                ")with(" +
                "   'connector' = 'filesystem', " +
                "   'path' = 'input/sensor.txt', " +
                "   'format' = 'csv' " +
                ")");

tEnv.sqlQuery("select * from sensor").execute().print();
```



## 事件时间：

事件时间属性可以用 .rowtime 后缀在定义 DataStream schema 的时候来定义。

**时间戳和 watermark 在这之前一定是在 DataStream 上已经定义好了。**

在从 DataStream 到 Table 转换时定义事件时间属性有两种方式:

### Table API：

```java
// 1. 添加一个字段作为事件时间
Table table = tEnv.fromDataStream(stream, $("id"), $("ts"), $("vc"), $("et").rowtime());
```

```java
// 2. 把现有字段直接变成事件时间
Table table = tEnv.fromDataStream(stream, $("id"), $("ts").rowtime(), $("vc"));
```

### SQL：

```java
tEnv.executeSql("create table sensor(" +
                "   id string, " +
                "   ts bigint, " +
                "   vc int, " +
                // 把一个数字转成一个时间戳类型: 参数1: 数字(毫秒或秒) 参数: 精度, 前面是s, 则填0, 是ms, 则填3
                "   et as TO_TIMESTAMP_LTZ(ts, 3),  " +// 添加一个时间戳字段, 能否直接认为他就是事件时间?  不能, 必须添加水印
                "   watermark for et as et - interval '3' second " + // 添加水印
                ")with(" +
                "   'connector' = 'filesystem', " +
                "   'path' = 'input/sensor.txt', " +
                "   'format' = 'csv' " +
                ")");
```

# 窗口

## Group Aggregation

> [Window Aggregation | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/sql/queries/window-agg/)

官方推荐使用TVF

## Windowing TVFs 

[Windowing TVF | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/sql/queries/window-tvf/)

### Tumble

```java
TUMBLE(TABLE data, DESCRIPTOR(timecol), size)
```

### Hop

```java
HOP(TABLE data, DESCRIPTOR(timecol), slide, size [, offset ])
```

### Cumulate

```java
CUMULATE(TABLE data, DESCRIPTOR(timecol), step, size)
```

案例：

```java
tEnv.executeSql("create table sensor(" +
                "   id string, " +
                "   ts bigint, " +
                "   vc int, " +
                "   et as TO_TIMESTAMP_LTZ(ts, 3),  " +// 添加一个时间戳字段, 能否直接认为他就是事件时间?  不能, 必须添加水印
                "   watermark for et as et - interval '3' second " + // 添加水印
                ")with(" +
                "   'connector' = 'filesystem', " +
                "   'path' = 'input/sensor.txt', " +
                "   'format' = 'csv' " +
                ")");
tEnv
    .sqlQuery("select " +
              " id, window_start, window_end, " +
              " sum(vc) " +
              
              // 滚动窗口
              "from table( tumble(table sensor, descriptor(et), interval '5' second) )" +
              
              // tvf 的滑动窗口: 限制窗口长度必须是滑动步长的整数倍
              "from table( hop(table sensor, descriptor(et), interval '2' second, interval '6' second) )" +
              
              // 定义了一个累积窗口:1. 窗口的步长(step): 窗口长度增长的单位;   2.  窗口的最大长度
              "from table( cumulate(table sensor, descriptor(et), interval '5' second, interval '20' second) )" +
              
              "group by id, window_start, window_end")  // 分组的时候: window_start, window_end 至少添加一个
    .execute()
    .print();
```



## Over Aggregation

### Table API

> [Table API | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/tableapi/#over-windows)

```java
Table result = table
    // define window
    .window(
        Over
          .partitionBy($("a"))
    	  //.orderBy("proctime")
          .orderBy($("rowtime"))
    	  //.preceding(UNBOUNDED_RANGE)
    	  //.preceding(lit(1).minutes())
    	  //.preceding(rowInterval(10))
          .preceding(lit(2).second())
          .following(CURRENT_RANGE)
          .as("w"))
    // sliding aggregate
    .select(
        $("a"),
        $("b").avg().over($("w")),
        $("b").max().over($("w")),
        $("b").min().over($("w").as("b_min"))
    );
```



### SQL

> [Over Aggregation | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/sql/queries/over-agg/)

```java
SELECT
  agg_func(agg_col) OVER (
    [PARTITION BY col1[, col2, ...]]
    ORDER BY time_col
    range_definition),
  ...
FROM ...
```

```java
tEnv
    .sqlQuery("select " +
              " *, " +
              " sum(vc) over(partition by id order by et rows between 1 preceding and current row)  as vc_sum " +
              // 默认是时间这种正交性
              " sum(vc) over(partition by id order by et range between unbounded preceding and current row)  as vc_sum " +
              " sum(vc) over(partition by id order by et range between interval '2' second preceding and current row)  as vc_sum " +
              " sum(vc) over(partition by id order by et)  as vc_sum " + 
              "from sensor")
    .execute()
    .print();
```

```java
//另一种写法：
tEnv
    .sqlQuery("select " +
              " *, " +
              " sum(vc) over w  as vc_sum, " +
              " max(vc) over w  as max_sum " +
              "from sensor " +
              "window w as(partition by id order by et rows between unbounded preceding and current row)")
    .execute()
    .print();
```

# 函数

## 内置函数

> [System (Built-in) Functions | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/functions/systemfunctions/#logical-functions)

### Scalar Functions

输入: 0个1个或多个

输出: 1个

### Aggregate Functions

## 自定义函数

> [User-defined Functions | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/functions/udfs/)

### Scalar Functions

定义函数：

```java
// 定义一个可以把字符串转成大写标量函数
public static class ToUpperCase extends ScalarFunction {
    public String eval(String s){
        return s.toUpperCase();
    }
}
```

在Table API中使用

```java
Table table = tEnv.fromDataStream(stream, $("id"), $("ts"), $("vc"));

// 1. table api 使用方式1: 不注册直接 inline 使用
// call: 参数1: 自定义函数的类型 参数2: 传给自定义函数的数据
table.select($("id"), call(MyUpper.class, $("id")).as("upper_case")).execute().print();

// 2. table api 使用方式2: 注册后使用
// 2.1 注册函数
tEnv.createTemporaryFunction("toUpper", ToUpperCase.class);
// 2.2 使用函数
table.select($("id"), call("my_upper", $("id")).as("upper_case")).execute().print();
```

在SQL中使用

```java
// 2. 在sql中使用, 必须先注册函数
tEnv.createTemporaryFunction("my_upper", MyUpper.class);
//注册临时表
tEnv.createTemporaryView("sensor", table);
tEnv.sqlQuery("select " +
              "   id, " +
              "   my_upper(id) id_upper " +
              " from sensor").execute().print();
```



### Table Functions

定义函数：

```java
@FunctionHint(output = @DataTypeHint("ROW(word string, len int)"))
public static class Split extends TableFunction<Row> {
    public void eval(String line) {
        if (line.length() == 0) {
            return;
        }
        for (String s : line.split(",")) {
            // 来一个字符串, 按照逗号分割, 得到多行, 每行为这个单词和他的长度
            collect(Row.of(s, s.length()));
        }
    }
}
```

在Table API中使用

```java

Table table = tEnv.fromDataStream(stream, $("line"));

// 1. 内联使用
table
    .joinLateral(call(Split.class, $("line")))
    .select($("line"), $("word"), $("len"))
    .execute()
    .print();

table
    .leftOuterJoinLateral(call(Split.class, $("line")))
    .select($("line"), $("word"), $("len"))
    .execute()
    .print();

// 2. 注册后使用
tEnv.createTemporaryFunction("split", Split.class);
table
    .joinLateral(call("split", $("line")))
    .select($("line"), $("word"), $("len"))
    .execute()
    .print();
```

在SQL中使用

```java
// 1. 注册表
tEnv.createTemporaryView("t_word", table);
// 2. 注册函数
tEnv.createTemporaryFunction("split", Split.class);
// 3. 使用函数
// 3.1 join
//内连接1: select  .. from a join b on a.id=b.id
tEnv.sqlQuery("select " +
                  " line, word, len " +
                  "from t_word " +
                  "join lateral table(split(line)) on true").execute().print();
// 或者

//内连接2: select  .. from a, b where a.id=b.id
//where true 可以省略
tEnv.sqlQuery("select " +
                  " line, word, len " +
                  "from t_word, " +
                  "lateral table(split(line))").execute().print();
// 3.2 left join
tEnv.sqlQuery("select " +
                  " line, word, len " +
                  "from t_word " +
                  "left join lateral table(split(line)) on true").execute().print();
// 3.3 join或者left join给字段重命名
tEnv.sqlQuery("select " +
                  " line, new_word, new_len " +
                  "from t_word " +
                  "left join lateral table(split(line)) as t(new_word, new_len) on true").execute().print();
```

![image-20220915233852354](https://gitee.com/liujiananliu/upload-image/raw/master/202209152338389.png)

### Aggregate Functions

定义函数：

必须实现的方法：

`createAccumulator()`

`accumulate(...)`

`getValue(...)`

```java
// 累加器类型
public static class VcAvgAcc {
    public Integer sum = 0;
    public Long count = 0L;
}

public static class VcAvg extends AggregateFunction<Double, VcAvgAcc> {
    
    // 返回最终的计算结果
    @Override
    public Double getValue(VcAvgAcc accumulator) {
        return accumulator.sum * 1.0 / accumulator.count;
    }
    
    // 初始化累加器
    @Override
    public VcAvgAcc createAccumulator() {
        return new VcAvgAcc();
    }
    
    // 处理输入的值, 更新累加器
    // 参数1: 累加器
    // 参数2,3,...: 用户自定义的输入值
    public void accumulate(VcAvgAcc acc, Integer vc) {
        acc.sum += vc;
        acc.count += 1L;
    }
}
```

在Table API中使用

```java
// 1. 内联使用
table
    .groupBy($("id"))
    .select($("id"), call(VcAvg.class, $("vc")))
    .execute()
    .print();

// 2. 注册后使用
tEnv.createTemporaryFunction("my_avg", VcAvg.class);
table
    .groupBy($("id"))
    .select($("id"), call("my_avg", $("vc")))
    .execute()
    .print();
```

在SQL中使用

```java
// 1. 注册表
tEnv.createTemporaryView("t_sensor", table);
// 2. 注册函数
tEnv.createTemporaryFunction("my_avg", VcAvg.class);
// 3. sql中使用自定义聚合函数
tEnv.sqlQuery("select id, my_avg(vc) from t_sensor group by id").execute().print();
```



### Table Aggregate Functions

定义函数：

必须实现的方法：

`createAccumulator()`

`accumulate(...)`

`emitValue(...)`

```java
// 累加器
public static class Top2Acc {
    public Integer first = Integer.MIN_VALUE; // top 1
    public Integer second = Integer.MIN_VALUE; // top 2
}

// Tuple2<Integer, Integer> 值和排序
public static class Top2 extends TableAggregateFunction<Tuple2<Integer, Integer>, Top2Acc> {
    
    @Override
    public Top2Acc createAccumulator() {
        return new Top2Acc();
    }
    
    public void accumulate(Top2Acc acc, Integer vc) {
        if (vc > acc.first) {
            acc.second = acc.first;
            acc.first = vc;
        } else if (vc > acc.second) {
            acc.second = vc;
        }
    }
    
    public void emitValue(Top2Acc acc, Collector<Tuple2<Integer, Integer>> out) {
        if (acc.first != Integer.MIN_VALUE) {
            out.collect(Tuple2.of(acc.first, 1));
        }
        
        if (acc.second != Integer.MIN_VALUE) {
            out.collect(Tuple2.of(acc.second, 2));
        }
        
    }
}
```

在Table API中使用

```java
// 1. 内联使用
table
    .groupBy($("id"))
    .flatAggregate(call(Top2.class, $("vc")).as("v", "rank"))
    .select($("id"), $("v"), $("rank"))
    .execute()
    .print();
// 2. 注册后使用
tEnv.createTemporaryFunction("top2", Top2.class);
table
    .groupBy($("id"))
    .flatAggregate(call("top2", $("vc")).as("v", "rank"))
    .select($("id"), $("v"), $("rank"))
    .execute()
    .print();
```

在SQL中使用

```java
//目前不支持
```



# topN

> [Window Top-N | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/sql/queries/window-topn/)

The following shows the syntax of the Window Top-N statement:

```java
SELECT [column_list]
FROM (
   SELECT [column_list],
     ROW_NUMBER() OVER (PARTITION BY window_start, window_end [, col_key1...]
       ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
   FROM table_name) -- relation applied windowing TVF
WHERE rownum <= N [AND conditions]
```

The following example shows how to calculate Top 3 suppliers who have the highest sales for every tumbling 10 minutes window.

```java
Flink SQL> SELECT *
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY price DESC) as rownum
    FROM (
      SELECT window_start, window_end, supplier_id, SUM(price) as price, COUNT(*) as cnt
      FROM TABLE(
        TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
      GROUP BY window_start, window_end, supplier_id
    )
  ) WHERE rownum <= 3;
```



自己的案例：每隔30min 统计最近 1hour的热门商品 top3, 并把统计的结果写入到mysql中

特殊的计算方式，必须按照步骤来

```java
Configuration conf = new Configuration();
conf.setInteger("rest.port", 2000);
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(conf);
env.setParallelism(1);
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);

// 1. 创建动态表与数据源(文件)关联, 定义时间属性, 后期用到了窗口
tEnv.executeSql("create table ub(" +
                "   user_id bigint, " +
                "   item_id bigint, " +
                "   category_id int, " +
                "   behavior string, " +
                "   ts bigint, " +
                "   et as to_timestamp_ltz(ts, 0), " +
                "   watermark for et as et - interval '3' second" +
                ")with(" +
                " 'connector' = 'filesystem', " +
                " 'path' = 'input/UserBehavior.csv', " +
                " 'format' = 'csv' " +
                ")");

// 2. 过滤出点击数据(pv), 统计每个商品在每个窗口内的点击量   tvf聚合  (流中的第一次keyBy)
Table t1 = tEnv.sqlQuery("select " +
                         " window_start, " +
                         " window_end, " +
                         " item_id, " +
                         " count(*) ct " +
                         " from  table( tumble( table ub, descriptor(et), interval '1' hour ) ) " +
                         " where  behavior='pv' " +
                         " group by window_start, window_end, item_id");
tEnv.createTemporaryView("t1", t1);

// 3. 使用over窗口,按照窗口(窗口结束时间)分组 按照点击量进行降序排序, 取名次   (流中的第二次keyBy)
// rank/dense_rank/row_number(仅支持)
Table t2 = tEnv.sqlQuery("select " +
                         "window_end, " +
                         "item_id, " +
                         "ct, " +
                         "row_number() over(partition by window_end order by ct desc) rn " +
                         "from t1");
tEnv.createTemporaryView("t2", t2);

// 4. 过滤出 top3  (where rn <= 3)
Table result = tEnv.sqlQuery("select " +
                             " window_end w_end, " +
                             " item_id, " +
                             " ct item_count, " +
                             " rn rk " +
                             "from t2 " +
                             "where rn<= 3");

// 5. 写出到mysql中
// 5.1 建一个动态表与mysql关联, 用jdbc连接器
tEnv.executeSql("CREATE TABLE hot_item ( " +
                "  w_end timestamp, " +
                "  item_id bigint, " +
                "  item_count bigint, " +
                "  rk bigint, " +
                "  PRIMARY KEY (w_end, rk) NOT ENFORCED " +
                ") WITH ( " +
                "   'connector' = 'jdbc', " +
                "   'url' = 'jdbc:mysql://hadoop162:3306/flink_sql?useSSL=false', " +
                "   'table-name' = 'hot_item', " +
                "   'username' = 'root', " +
                "   'password' = 'aaaaaa' " +
                ")");

result.executeInsert("hot_item");
```

