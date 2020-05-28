## 消费者`offset`的存储
我们在 [查看Consumer消费的情况.md](查看Consumer消费的情况.md) 中说过，我们可以使用`kafka-consumer-groups.sh`脚本可以查看消费者消费消息的详细信息，包括消费者消费到的`offset`，而且当这个消费者挂了的时候，我们还是可以通过上面的脚本来查询消费者的`offset`，下面是讲`reset offset`时候的截图：
![image-20200527103529118](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527103529.png)

可以看出：当消费者挂了的时候，我们还是可以通过`kafka-consumer-groups.sh`脚本来查询消费者的`offset`。其实当Kafka集群重启后，我们也可以通过`kafka-consumer-groups.sh`脚本来查询消费者的`offset`

那么上面的现象可以说明：消费者消费到的`offset`肯定是存储到了Kafka集群的某个文件中，要不然重启Kafka集群肯定看不到之前消费者的`offset`了

从Kafka的`0.9.x`版本开始，每一个消费者消费到的`offset`都会存储到一个名为`__consumer_offsets`的`topic`中，我们现在可以通过如下的命令来看下，确实存在这个`topic`：
```shell
    ## 在任何一台机器上执行下面的命令
    cd ~/bigdata/kafka_2.11-1.0.0
    bin/kafka-topics.sh --list --zookeeper master:2181
    
    ## 也可以通过这个命令去查看这个topic的详细信息
    bin/kafka-topics.sh --describe --zookeeper master:2181 --topic __consumer_offsets
```
这个名为`__consumer_offsets`的`topic`就是一个普通的`topic`，和其他的`topic`都一样，`topic`该有特性，这个`__consumer_offsets`也都有，只不过这个`__consumer_offsets`存储的数据是每一个消费者消费到的`offset`消息而已

> 其实，早在 0.8.2.2 版本，已支持存入消费的 offset 到Topic中，只是那时候默认是将消费的 offset 存放在 Zookeeper 集群中。那现在，官方默认将消费的offset存储在 Kafka 的Topic中，同时，也保留了存储在 Zookeeper 的接口，通过 offsets.storage=zookeeper 属性来进行设置。

> 如果消费者消费的offset是存储在zookeeper中的话，那么就需要通过下面的命令来查看消费者消费的详细信息了：`bin/kafka-consumer-groups.sh --zookeeper master:2181 --describe --group group1`

## 消费者存储`offset`的时间点
我们在讲[Kafka消息传递语义]( [Kafka的ack机制.md](Kafka的ack机制.md) )的时候，讲过了Kafka默认的语义是`At least once`语义，也就是说在`Consumer`消费消息的时候是这样的：先读取消息，然后处理消息，然后保存处理结果，最后就是保存消费到的消息的`offset`，我们看下面很简单的消费者代码：

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
        // 轮询的去消费topic中的消息
        while (true) {
            // 这行代码干了两件事
            // 1、保存上一次轮询消费的消息的offset到kafka中名为__consumer_offsets的topic中
            // 2、拉取当前这次轮询的消息
            ConsumerRecords<String, String> records = consumer.poll(100);
            // 处理消息
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s, topic = %s, partition = %d",
                        record.offset(), record.key(), record.value(), record.topic(), record.partition());
                System.out.println();
            }
            // 当然，这里可以保存处理结果，比如 save2DB等操作
        }
    }
}
```

如果我们想实现`At most once`语义的话，我们可以自己控制`offset`的保存，即流程为：先读取消息，然后保存消费到的消息的`offset`，然后处理消息，最后就是保存处理结果
如下代码：
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
        
        // 设置不使用默认的自动提交保存offset的特性
        props.put("enable.auto.commit", "false") 

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(Arrays.asList("test-group"));
        // 轮询的去消费topic中的消息
        while (true) {
            // 1、这行代码现在就不会自动的保存上一次轮询消费的消息的offset
            // 2、拉取当前这次轮询的消息
            ConsumerRecords<String, String> records = consumer.poll(100);
            
            // 保存消费的offset
            // 异步保存offset
            consumer.commitAsync();
            // 也可以同步保存offset
            //consumer.commitSync();
            
            // 处理消息
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s, topic = %s, partition = %d",
                        record.offset(), record.key(), record.value(), record.topic(), record.partition());
                System.out.println();
            }
            // 当然，这里可以保存处理结果，比如 save2DB等操作
            
             
        }
    }
}
```

Kafka常见API的详细讲解请参考： [kafka常用api.md](kafka常用api.md) 