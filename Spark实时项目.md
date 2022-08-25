## 开发清单

1. 创建父工程，添加依赖
   ```xml
      <!-- properties定义一些变量：在pom文件中可以使用${标签名}获取标签中的值,举例: ${spark.version} = 3.0.0 -->
       <properties>
           <spark.version>3.0.0</spark.version>
   		...
       </properties>
   
       <!-- dependencies中编写的dependency，会被module无条件继承（无需显式声明，就拥有） -->
       <dependencies>
           <dependency>
   			...
           </dependency>
           ...
       </dependencies>
   
       <!-- dependencyManagement 中编写的dependency,需要module显式声明（无需声明version）后，才会继承 -->
       <dependencyManagement>
           <dependencies>
               <dependency>
   				...
               </dependency>
   			...
           </dependencies>
       </dependencyManagement>
   ```

   

2. 创建Common模块

   - 提供常量

   - 准备连接所有数据库环境的配置文件

   - 提供一个工具类，用户读取配置文件中指定key的value
   - 提供一个工具类，可以向kafka中去生产数据
   - 提供一个工具类，可以返回一个Jedis客户端

3. 能够模拟数据到Kafka

4. 编写SparkRealtimeStreaming模块，进行ETL处理

   - 编写App，可以读取写入到kafka的主题中的数据，挑选出其中的startlog和actionslog
   - 将startlog扁平化写回到kafka
   - 将actionsLog炸裂写回到kafka