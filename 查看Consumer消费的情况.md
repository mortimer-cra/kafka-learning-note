在使用`Consumer Group`的时候，我们经常会碰到想去查看：
1. 查看有哪些消费组
2. 每一个消费组中的消费者消费消息的状态是怎么样的？
3. 对消费者进行`reset offset`

上面的事情我们都可以使用`bin/kafka-consumer-groups.sh`这个脚本来做到

### 查看有哪些消费组
我们可以通过下面的命令查看现在有多少的消费组：
```shell
cd /home/hadoop-twq/bigdata/kafka_2.11-1.0.0
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --list
```
当你执行上面的命令的时候，不会有任何的输出

现在现在还没有任何的消费组消费`Kafka`集群的消息。我们尝试在本地执行下面的消费消息的Java代码：
```java
package com.twq.streaming.kafka.example;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class SimpleComsumerGroup1 {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "group1");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(Arrays.asList("test-group"));


        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s, topic = %s, partition = %d",
                        record.offset(), record.key(), record.value(), record.topic(), record.partition());
                System.out.println();
            }
        }
    }
}
```
以上代码依赖的`kafka-clients`的版本是`1.0.0`，如下：
```xml
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>1.0.0</version>
    </dependency>
```
启动完上面的消费者后，我们可以通过下面的命令查看现在有多少的消费组：
```shell
cd /home/hadoop-twq/bigdata/kafka_2.11-1.0.0
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --list
```
输出结果为：
```
group1
```
即表示，目前只有一个名称为`group1`的消费组。

### 每一个消费组中的消费者消费消息的状态是怎么样的？
我们现在可以使用下面的命令来查看`group1`这个消费组的详细信息：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --describe --group group1
```
这个命令就是查询出`group1`这个消费组包含多少个`comsumer`，么一个消费者消费消息的详细信息是什么，如下是输出结果：
```
TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID                                       HOST            
test-group  0          0               0               0    consumer-1-3e83792d-7328-43aa-ba69-c3d1809cc384   /192.168.126.1  


CLIENT-ID
consumer-1
```
从上面可以看出：

1. 消费组`group1`中目前只包含一个消费者
2. TOPIC：这个消费者消费的是`test-group`这个topic中的消息
3. PARTITION：表示这个消费者消费的是第一个分区的消息
3. CURRENT-OFFSET：这个消费者目前消费的消息的offset是`0`，其实就是表明还没开始消费消息
4. LOG-END-OFFSET：消费者消费的这个分区的最大的`offset`，这里也是`0`，表明这个`test-group`中还没有数据(注意：我这个topic的消息已经存储超过`168`个小时了，所以数据都被清除了，你那边可能还有数据)
5. LAG：这个是等于`LOG-END-OFFSET - CURRENT-OFFSET`，表示这个消费者消费消息的滞后数
6. CONSUMER-ID：表示这个consumer的id
7. HOST：表示这个消费者所在机器的ip
8. CLIENT-ID：消费者所在的客户端的id


我们尝试使用下面的命令，给`test-group`发送2条消息，如下：
```shell
bin/kafka-console-producer.sh --broker-list master:9092 --topic test-group
```

然后我们现在可以使用下面的命令来查看`group1`这个消费组的详细信息：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --describe --group group1
```
输出结果如下：
```
TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID                                       HOST            
test-group  0          2               2               0    consumer-1-3e83792d-7328-43aa-ba69-c3d1809cc384   /192.168.126.1  

CLIENT-ID
consumer-1
```
表示这个消费者已经消费到`offset=2`的消息了

### 对消费者进行`reset offset`
我们现在停止掉上面的Java代码(就是停止掉消费者)，然后可以继续执行下面的命令来查看`group1`这个消费组的详细信息：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --describe --group group1
```
输出结果如下：
```
TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID                                       HOST            
test-group  0          2               2               0    consumer-1-3e83792d-7328-43aa-ba69-c3d1809cc384   /192.168.126.1  

CLIENT-ID
consumer-1
```
现在我们执行下面的命令将消费者组`group1`中所有的消费者消费的消息的`offset`重置到最早的消息的`offset`
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --group group1 --reset-offsets --to-earliest --all-topics --execute
```
输出结果为：
```
TOPIC                          PARTITION  NEW-OFFSET     
test-group                     0          0       
```
即表明，我们已经将消费者消费的消息的`offset`重置为最早的消息的`offset`了。如下：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --describe --group group1
```
输出结果如下：
```
TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID  HOST  CLIENT-ID
test-group  0          0               2               2    -            -     -
```
上面的输出说明还没有开始消费消息了，当我们再次启动上面Java代码的消费者，那这个消费者肯定要从`offset=0`开始消费了。(你可以再执行下上面的Java代码看看)，如下：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --describe --group group1
```
输出结果如下：
```shell
TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID                                       HOST            
test-group  0          2               2               0    consumer-1-ce669eec-16dd-42f0-acc0-7c6b354783fd   /192.168.126.1  

CLIENT-ID
consumer-1
```
以上其实就是消费者消费的`offset`的重置了，这里一定要注意**重置offset的前提必须是consumer group必须是inactive的，即不能是处于正在工作中的状态**

我们重置一个消费组的`offset`的命令是：
```shell
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --group group1 --reset-offsets --to-earliest --all-topics --execute
```
其中：
1. 除了`--to-earliest即把位移调整到分区当前最小位移`，还支持：
    1. --to-latest：把位移调整到分区当前最新位移
    2. --to-current：把位移调整到分区当前位移
    3. --to-offset <offset>： 把位移调整到指定位移处
    4. --shift-by N： 把位移调整到当前位移 + N处，注意N可以是负数，表示向前移动
    5. --to-datetime <datetime>：把位移调整到大于给定时间的最早位移处，datetime格式是yyyy-MM-ddTHH:mm:ss.xxx，比如2017-08-04T00:00:00.000
    6. --by-duration <duration>：把位移调整到距离当前时间指定间隔的位移处，duration格式是PnDTnHnMnS，比如PT0H5M0S
    7. --from-file <file>：从CSV文件中读取调整策略
2. 除了使用`--all-topics即为consumer group下所有topic的所有分区调整位移`，还支持：
    1. --topic t1 --topic t2（为指定的若干个topic的所有分区调整位移）
    2. --topic t1:0,1,2（为指定的topic分区调整位移）
3. 除了使用`--execute即执行真正的位移调整`
    1. 什么参数都不加：只是打印出位移调整方案，不具体执行
    2. --export：把位移调整方案按照CSV格式打印，方便用户成csv文件，供后续直接使用

```shell
## 下面表示将所有分区位移调整为30分钟之前的最早位移
bin/kafka-consumer-groups.sh --bootstrap-server master:9092 --group test-group --reset-offsets --all-topics --by-duration PT0H30M0S
```

### 脚本`kafka-consumer-groups.sh`的详细详细参数如下
```shell
[hadoop-twq@master kafka_2.11-1.0.0]$ bin/kafka-consumer-groups.sh
List all consumer groups, describe a consumer group, delete consumer group info, or reset consumer group offsets.
Option                                  Description                            
------                                  -----------                            
--all-topics                            Consider all topics assigned to a      
                                          group in the `reset-offsets` process.
--bootstrap-server <String: server to   REQUIRED (for consumer groups based on 
  connect to>                             the new consumer): The server to     
                                          connect to.                          
--by-duration <String: duration>        Reset offsets to offset by duration    
                                          from current timestamp. Format:      
                                          'PnDTnHnMnS'                         
--command-config <String: command       Property file containing configs to be 
  config property file>                   passed to Admin Client and Consumer. 
--delete                                Pass in groups to delete topic         
                                          partition offsets and ownership      
                                          information over the entire consumer 
                                          group. For instance --group g1 --    
                                          group g2                             
                                        Pass in groups with a single topic to  
                                          just delete the given topic's        
                                          partition offsets and ownership      
                                          information for the given consumer   
                                          groups. For instance --group g1 --   
                                          group g2 --topic t1                  
                                        Pass in just a topic to delete the     
                                          given topic's partition offsets and  
                                          ownership information for every      
                                          consumer group. For instance --topic 
                                          t1                                   
                                        WARNING: Group deletion only works for 
                                          old ZK-based consumer groups, and    
                                          one has to use it carefully to only  
                                          delete groups that are not active.   
--describe                              Describe consumer group and list       
                                          offset lag (number of messages not   
                                          yet processed) related to given      
                                          group.                               
--execute                               Execute operation. Supported           
                                          operations: reset-offsets.           
--export                                Export operation execution to a CSV    
                                          file. Supported operations: reset-   
                                          offsets.                             
--from-file <String: path to CSV file>  Reset offsets to values defined in CSV 
                                          file.                                
--group <String: consumer group>        The consumer group we wish to act on.  
--list                                  List all consumer groups.              
--new-consumer                          Use the new consumer implementation.   
                                          This is the default, so this option  
                                          is deprecated and will be removed in 
                                          a future release.                    
--reset-offsets                         Reset offsets of consumer group.       
                                          Supports one consumer group at the   
                                          time, and instances should be        
                                          inactive                             
                                        Has 3 execution options: (default) to  
                                          plan which offsets to reset, --      
                                          execute to execute the reset-offsets 
                                          process, and --export to export the  
                                          results to a CSV format.             
                                        Has the following scenarios to choose: 
                                          --to-datetime, --by-period, --to-    
                                          earliest, --to-latest, --shift-by, --
                                          from-file, --to-current. One         
                                          scenario must be choose              
                                        To define the scope use: --all-topics  
                                          or --topic. . One scope must be      
                                          choose, unless you use '--from-file' 
                                          scenario                             
--shift-by <Long: number-of-offsets>    Reset offsets shifting current offset  
                                          by 'n', where 'n' can be positive or 
                                          negative                             
--timeout <Long: timeout (ms)>          The timeout that can be set for some   
                                          use cases. For example, it can be    
                                          used when describing the group to    
                                          specify the maximum amount of time   
                                          in milliseconds to wait before the   
                                          group stabilizes (when the group is  
                                          just created, or is going through    
                                          some changes). (default: 5000)       
--to-current                            Reset offsets to current offset.       
--to-datetime <String: datetime>        Reset offsets to offset from datetime. 
                                          Format: 'YYYY-MM-DDTHH:mm:SS.sss'    
--to-earliest                           Reset offsets to earliest offset.      
--to-latest                             Reset offsets to latest offset.        
--to-offset <Long: offset>              Reset offsets to a specific offset.    
--topic <String: topic>                 The topic whose consumer group         
                                          information should be deleted or     
                                          topic whose should be included in    
                                          the reset offset process. In `reset- 
                                          offsets` case, partitions can be     
                                          specified using this format: `topic1:
                                          0,1,2`, where 0,1,2 are the          
                                          partition to be included in the      
                                          process. Reset-offsets also supports 
                                          multiple topic inputs.               
--zookeeper <String: urls>              REQUIRED (for consumer groups based on 
                                          the old consumer): The connection    
                                          string for the zookeeper connection  
                                          in the form host:port. Multiple URLS 
                                          can be given to allow fail-over.
```