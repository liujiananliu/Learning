# kafka source

```java
public class Flink03_Source_Kafka_1 {
    public static void main(String[] args) {
        
        Configuration conf = new Configuration();
        conf.setInteger("rest.port", 2000);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(conf);
        env.setParallelism(1);
        
        KafkaSource<String> source = KafkaSource
            .<String>builder()
            .setBootstrapServers("hadoop162:9092,hadoop163:9092,hadoop164:9092")
            .setGroupId("atguigu")
            .setTopics("s1")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();
        env
            .fromSource(source, WatermarkStrategy.noWatermarks(), "kafka_1")
            .print();
        
        
        try {
            env.execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# jdbc sink

```java
public class Flink06_Sink_Jdbc {
    public static void main(String[] args) {
        Configuration conf = new Configuration();
        conf.setInteger("rest.port", 2000);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(conf);
        env.setParallelism(1);

        env
            .socketTextStream("hadoop162", 9999)
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
            
            .keyBy(WaterSensor::getId)  // 方法引用
            .sum("vc")
            .addSink(JdbcSink.sink(
                "replace into sensor(id, ts, vc)values(?,?,?)",
                new JdbcStatementBuilder<WaterSensor>() {
                    @Override
                    public void accept(PreparedStatement ps,
                                       WaterSensor ws) throws SQLException {
                        // 给sql中的占位符赋值. 千万不要关闭ps, ps要重用
                        ps.setString(1, ws.getId());
                        ps.setLong(2, ws.getTs());
                        ps.setInt(3, ws.getVc());
                    }
                },
                new JdbcExecutionOptions.Builder()
                    .withBatchIntervalMs(1000)
                    .withBatchSize(1024)
                    .withMaxRetries(3)
                    .build(),
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                    .withDriverName("com.mysql.cj.jdbc.Driver")
                    .withUrl("jdbc:mysql://hadoop162:3306/test?useSSL=false")
                    .withUsername("root")
                    .withPassword("aaaaaa")
                    .build()
            ));
        
        try {
            env.execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# custom sink

```java
public class Flink05_Sink_Custom {
    public static void main(String[] args) {
        Configuration conf = new Configuration();
        conf.setInteger("rest.port", 2000);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(conf);
        env.setParallelism(1);
        
        env
            .socketTextStream("hadoop162", 9999)
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
            
            .keyBy(WaterSensor::getId)  // 方法引用
            .sum("vc")
            .addSink(new MySqlSink());
        
        try {
            env.execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public static class MySqlSink extends RichSinkFunction<WaterSensor>{
    
        private Connection connection;
    
        @Override
        public void open(Configuration parameters) throws Exception {
            // 获取mysql 连接
            // 1. 加载驱动
            Class.forName("com.mysql.jdbc.Driver");
            // 2. 获取连接 提升成员变量快捷键: alt+ctrl+f
            connection = DriverManager.getConnection("jdbc:mysql://hadoop162:3306/test?useSSL=false", "root", "aaaaaa");
        }
    
        @Override
        public void close() throws Exception {
            if (connection != null) {
                connection.close();
            }
        }
    
        @Override
        public void invoke(WaterSensor ws, Context context) throws Exception {
            // 插入数据
            // sql语句
//            String sql = "insert into sensor(id, ts, vc)values(?,?,?)";
            
            // 如果主键不重复就新增, 重复就更新
//            String sql = "insert into sensor(id, ts, vc)values(?,?,?) on duplicate key update vc=?";
            String sql = "replace into sensor(id, ts, vc)values(?,?,?)";  // 等价上面这个
            PreparedStatement ps = connection.prepareStatement(sql);
            // 给sql中的占位符赋值
            ps.setString(1, ws.getId());
            ps.setLong(2, ws.getTs());
            ps.setInt(3, ws.getVc());
//            ps.setInt(4, ws.getVc());
            
            ps.execute();
//            connection.commit();  // 提交执行. 当自动提交是true的时候,不能调用这个方法. mysql的自动提交就是true
            ps.close();
        }
    }
}
```

# 代码中如何添加水印

# 采集增量数据的工具：

maxwell

canal

