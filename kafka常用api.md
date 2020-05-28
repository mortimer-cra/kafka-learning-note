## 概述
随着`Kafka`版本的不断更新，其常用的`API`也是在不断的发展

1. 不管哪个版本，`Producer API`和`Consumer API`一直是存在的，这两种API才是我们经常会使用到的，接下来的内容也主要是围绕这两种API
2. 对于`Consumer API`，在版本`0.10.x`之前又分为两种：`高级消费API`和`低级消费API`。在`0.10.X`版本以及以后的版本中，这两种API被统一为`Consumer API`了，即不存在`高级消费API`和`低级消费API`的说法了
3. 从`0.10.X`版本开始，API的种类趋于稳定了，都包含了`Producer`、`Consumer`、`Streams`、`Connect`以及`AdminClient` 五类API

我们使用：

1. `0.9.X`版本的Kafka说明`高级消费API`和`低级消费API`
2. `1.0.0`版本的Kafka说明目前比较新的`Producer API`以及`Consumer API`

## `Producer API`
- `Producer API`主要的功能就是将消息发往Kafka集群，`Producer API`的使用非常的简单，基本所有的Kafka版本的`Producer API`都是一样的，就是调用`KafkaProducer`中的`send`方法，使用代码如下：
```java
    import org.apache.kafka.clients.producer.KafkaProducer;
    import org.apache.kafka.clients.producer.Producer;
    import org.apache.kafka.clients.producer.ProducerRecord;

    import java.util.Properties;

    public class SimpleProducer {
        public static void main(String[] args) {
            Properties props = new Properties();
        
            props.put("bootstrap.servers", "master:9092,slave1:9092,slave2:9092");
         
            props.put("acks", "all");
            props.put("retries", 0);
            
            props.put("batch.size", 16384);
            props.put("linger.ms", 1);
            props.put("buffer.memory", 33554432);
            
            props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
            props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    
            Producer<String, String> producer = new KafkaProducer<>(props);
            for (int i = 0; i < 100; i++) {
                // 一定要注意：这里的发送是异步发送的，产生的消息发送到Producer端的内存缓存中就返回，
                // 接下来都是由Producer端的Sender来完成将消息发送给broker server
                producer.send(new ProducerRecord<String, String>("my-topic",
                        Integer.toString(i), Integer.toString(i)));
            }
            producer.close();
        }
    }
```
- `Producer`端除了消息发送的功能，从`0.11.0.0`开始也支持了事务了，所以`Producer`端还有事务相关的API了，详细请参考： [Kafka生产者事务详解.md](Kafka生产者事务详解.md) 
- `Producer`端还可以查询指定`topic`的所有分区信息：
```java
// 查询指定topic的所有的分区的信息
List<PartitionInfo> partitionInfos = producer.partitionsFor("topic");

public class PartitionInfo {
    private final String topic; // 指定的topic
    private final int partition; // 这个topic某个partition
    private final Node leader; // 这个topic某个partition的Leader副本
    private final Node[] replicas; // 这个topic某个partition的所有副本
    private final Node[] inSyncReplicas; // 这个topic某个partition所有存活的副本
    private final Node[] offlineReplicas; // 这个topic某个partition所有挂了的副本
}
```
## `Consumer API`
### `0.9.X`版本消费者API
#### 低级消费者API(SimpleConsumer)
以下是使用低级消费者的API来消费指定`topic`的指定`partition`的消息的代码，主要的步骤有：
1. 查询到指定topic的指定partition的元数据信息
2. 查询到需要消费这个topic的指定partition的offset
3. 请求消费消息，如果有错误的话还需要自己处理错误
4. 如果`Leader`副本变了的话，还需要重新查询`Leader`副本

代码如下：
```java
package com.twq.low;

import kafka.api.FetchRequest;
import kafka.api.FetchRequestBuilder;
import kafka.api.PartitionOffsetRequestInfo;
import kafka.common.ErrorMapping;
import kafka.common.TopicAndPartition;
import kafka.javaapi.*;
import kafka.javaapi.consumer.SimpleConsumer;
import kafka.message.MessageAndOffset;

import java.nio.ByteBuffer;
import java.util.*;

public class SimpleExample {
    public static void main(String args[]) {
        SimpleExample example = new SimpleExample();
        long maxReads = Long.parseLong(args[0]); // 需要读取的消息的条数
        String topic = args[1]; // 消费的topic
        int partition = Integer.parseInt(args[2]); // 消费的分区
        List<String> seeds = new ArrayList<String>();
        seeds.add(args[3]); // 查找元数据信息的broker Server
        int port = Integer.parseInt(args[4]); // broker server监听的端口
        try {
            example.run(maxReads, topic, partition, seeds, port);
        } catch (Exception e) {
            System.out.println("Oops:" + e);
            e.printStackTrace();
        }
    }

    private List<String> replicaBrokers = null;

    public SimpleExample() {
        this.replicaBrokers = new ArrayList<String>();
    }

    public void run(long maxReads, String topic, int partition,
                    List<String> seedBrokers, int port) throws Exception {
        // 1、查询到指定topic的指定partition的元数据信息
        PartitionMetadata metadata = findPartitionMetadata(seedBrokers, port, topic, partition);
        if (metadata == null) { // 找不到这个分区
            System.out.println("Can't find metadata for Topic and Partition. Exiting");
            return;
        }
        if (metadata.leader() == null) { // 这个分区没有leader副本
            System.out.println("Can't find Leader for Topic and Partition. Exiting");
            return;
        }
        String leadBroker = metadata.leader().host();
        String clientName = "Client_" + topic + "_" + partition;

        SimpleConsumer consumer =
                new SimpleConsumer(leadBroker, port, 100000, 64 * 1024, clientName);

        // 2、查询到需要消费这个topic的指定partition的offset
        long readOffset = getLastOffset(consumer,topic, partition, kafka.api.OffsetRequest.EarliestTime(), clientName);

        // 3、开始读取消息
        int numErrors = 0;
        while (maxReads > 0) {
            if (consumer == null) {
                consumer = new SimpleConsumer(leadBroker, port, 100000, 64 * 1024, clientName);
            }
            // 3.1、构建读取消息的请求，并请求拉取消息
            FetchRequest req = new FetchRequestBuilder()
                    .clientId(clientName)
                    .addFetch(topic, partition, readOffset, 100000) // Note: this fetchSize of 100000 might need to be increased if large batches are written to Kafka
                    .build();
            FetchResponse fetchResponse = consumer.fetch(req);

            // 3.2、错误处理
            if (fetchResponse.hasError()) { // 如果返回有错误的话
                numErrors++;
                // 拿到错误code
                short code = fetchResponse.errorCode(topic, partition);
                System.out.println("Error fetching data from the Broker:" + leadBroker + " Reason: " + code);
                if (numErrors > 5) break; // 如果错了5次的话，则不读数据了
                if (code == ErrorMapping.OffsetOutOfRangeCode())  {
                    // 如果请求的offset不对，则再次获取一个正常的offset，重新读数据
                    readOffset = getLastOffset(consumer,topic, partition,
                            kafka.api.OffsetRequest.LatestTime(), clientName);
                    continue;
                }
                consumer.close();
                consumer = null;
                // 可能是因为leader副本挂了导致的失败，所以重新在查询一次新的leader副本
                leadBroker = findNewLeader(leadBroker, topic, partition, port);
                continue;
            }
            numErrors = 0;

            long numRead = 0;
            // 3.3、读取这个topic的指定partition的消息
            for (MessageAndOffset messageAndOffset : fetchResponse.messageSet(topic, partition)) {
                long currentOffset = messageAndOffset.offset(); // 当前的offset
                if (currentOffset < readOffset) {
                    System.out.println("Found an old offset: " + currentOffset + " Expecting: " + readOffset);
                    continue;
                }
                readOffset = messageAndOffset.nextOffset(); // 下一个读取的offset
                ByteBuffer payload = messageAndOffset.message().payload(); // 当前消息的内容

                // 转换消息并且打印
                byte[] bytes = new byte[payload.limit()];
                payload.get(bytes);
                System.out.println(String.valueOf(messageAndOffset.offset()) + ": " + new String(bytes, "UTF-8"));
                numRead++;
                maxReads--;
            }

            if (numRead == 0) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        if (consumer != null) consumer.close();
    }

    /**
     * 查询新的指定分区的Leader副本
     * @param oldLeader 已经挂掉了的leader副本
     * @param topic 指定主题
     * @param partition 指定分区
     * @param port broker监听端口
     * @return
     * @throws Exception
     */
    private String findNewLeader(String oldLeader, String topic, int partition, int port) throws Exception {
        // 尝试3次来查询，如果3次都没有找到leader副本的话，则抛出异常
        for (int i = 0; i < 3; i++) {
            boolean goToSleep = false;
            PartitionMetadata metadata = findPartitionMetadata(replicaBrokers, port, topic, partition);
            if (metadata == null) {
                goToSleep = true;
            } else if (metadata.leader() == null) {
                goToSleep = true;
            } else if (oldLeader.equalsIgnoreCase(metadata.leader().host()) && i == 0) {
                // first time through if the leader hasn't changed give ZooKeeper a second to recover
                // second time, assume the broker did recover before failover, or it was a non-Broker issue
                //
                goToSleep = true;
            } else {
                return metadata.leader().host();
            }
            if (goToSleep) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        System.out.println("Unable to find new leader after Broker failure. Exiting");
        throw new Exception("Unable to find new leader after Broker failure. Exiting");
    }

    /**
     *  找到需要读取的消息的开始的offset
     * @param consumer 消费者
     * @param topic 指定topic
     * @param partition 指定分区
     * @param whichTime 从哪个时间点开始消费消息
     * @param clientName 消费客户端名称
     * @return 指定topic、指定分区、指定时间点的offset
     */
    public static long getLastOffset(SimpleConsumer consumer, String topic, int partition,
                                     long whichTime, String clientName) {
        TopicAndPartition topicAndPartition = new TopicAndPartition(topic, partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo =
                new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        requestInfo.put(topicAndPartition, new PartitionOffsetRequestInfo(whichTime, 1));
        kafka.javaapi.OffsetRequest request =
                new kafka.javaapi.OffsetRequest(requestInfo, kafka.api.OffsetRequest.CurrentVersion(),clientName);
        OffsetResponse response = consumer.getOffsetsBefore(request);

        if (response.hasError()) {
            System.out.println("Error fetching data Offset Data the Broker. Reason: " + response.errorCode(topic, partition) );
            return 0;
        }
        long[] offsets = response.offsets(topic, partition);
        return offsets[0];
    }

    /**
     *  为指定topic的指定partition找到leader副本所在的broker
     * @param seedBrokers
     * @param port
     * @param topic
     * @param partition
     * @return
     */
    private PartitionMetadata findPartitionMetadata(List<String> seedBrokers, int port, String topic, int partition) {
        for (String seed : seedBrokers) {
            SimpleConsumer consumer =
                    new SimpleConsumer(seed, port, 100000, 64 * 1024, "leaderLookup");
            try {
                List<String> topics = Collections.singletonList(topic);
                // 发起查询topic元数据信息的请求
                TopicMetadataRequest req = new TopicMetadataRequest(topics);
                kafka.javaapi.TopicMetadataResponse resp = consumer.send(req);

                // 拿到topic的元数据信息
                List<TopicMetadata> metaData = resp.topicsMetadata();
                for (TopicMetadata item : metaData) {
                    // 循环遍历这个topic的所有的分区元数据信息
                    for (PartitionMetadata part : item.partitionsMetadata()) {
                        // 找到相对应的分区
                        if (part.partitionId() == partition) {
                            replicaBrokers.clear();
                            for (kafka.cluster.BrokerEndPoint replica : part.replicas()) {
                                // 将这个分区的所有的副本都放到成员变量replicaBrokers中
                                replicaBrokers.add(replica.host());
                            }
                            return part;
                        }
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + seed + "] to find Leader for [" + topic
                        + ", " + partition + "] Reason: " + e);
            } finally {
                if (consumer != null) consumer.close();
            }
        }
        return null;
    }
}
```
参考：[SimpleConsumer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)

上面的代码看起来比较困难，不过没关系，你只需要记住使用低级消费者API的目的：主要是可以自己控制分区消息的消费情况，比如：
1. 可以多次的读取某一条消息
2. 可以只消费一个`topic`中的一个或者几个分区的消息
3. 还可以控制使得一条消息只处理一次

我们使用低级消费者API，虽然在消息的消费上有很大的控制权，但是呢，低级消费者API也有不少的缺点，比如：
1. 你必须自己跟踪消费者消费的offset
2. 你必须自己找到指定`topic`的指定`partition`的`Leader`副本所在的broker
3. 当`Leader`副本变化了的时候，你必须要重新找到新的`Leader`副本
4. 大部分的消费的错误，你还需要自己去处理

#### 高级消费者API
很多的场景，消费Kafka中的消息的时候，其实就是关心Kafka中的消息，而不太关系消费的offset了，所以使用高级消费的API就可以让你不用关心消费的offset了，代码如下：
```java
package com.twq.high;

import kafka.consumer.ConsumerConfig;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ConsumerGroupExample {
    private final ConsumerConnector consumer;
    private final String topic;
    private ExecutorService executor;

    public ConsumerGroupExample(String zookeeper, String groupId, String topic) {
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(
                createConsumerConfig(zookeeper, groupId));
        this.topic = topic;
    }

    public void shutdown() {
        if (consumer != null) consumer.shutdown();
        if (executor != null) executor.shutdown();
        try {
            if (!executor.awaitTermination(5000, TimeUnit.MILLISECONDS)) {
                System.out.println("Timed out waiting for consumer threads to shut down, exiting uncleanly");
            }
        } catch (InterruptedException e) {
            System.out.println("Interrupted during shutdown, exiting uncleanly");
        }
    }

    public void run(int numThreads) {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(numThreads));
        // 消费指定topic的消息
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);

        // 启动指定数量的线程来消费消息
        executor = Executors.newFixedThreadPool(numThreads);
        int threadNumber = 0;
        for (final KafkaStream stream : streams) {
            executor.submit(new ConsumerTest(stream, threadNumber));
            threadNumber++;
        }
    }

    private static ConsumerConfig createConsumerConfig(String zookeeper, String groupId) {
        Properties props = new Properties();
        props.put("zookeeper.connect", zookeeper);
        props.put("group.id", groupId);
        props.put("zookeeper.session.timeout.ms", "400");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");

        return new ConsumerConfig(props);
    }

    public static void main(String[] args) {
        String zooKeeper = args[0]; // zk的位置
        String groupId = args[1]; // groupId
        String topic = args[2]; // topic
        int threads = Integer.parseInt(args[3]); // 处理消息的线程数

        ConsumerGroupExample example = new ConsumerGroupExample(zooKeeper, groupId, topic);
        example.run(threads);

        try {
            Thread.sleep(10000);
        } catch (InterruptedException ie) {

        }
        example.shutdown();
    }
}
```
```java
package com.twq.high;

import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;

public class ConsumerTest implements Runnable {
    private KafkaStream stream;
    private int threadNumber;

    public ConsumerTest(KafkaStream stream, int threadNumber) {
        this.threadNumber = threadNumber;
        this.stream = stream;
    }

    public void run() {
        ConsumerIterator<byte[], byte[]> it = stream.iterator();
        while (it.hasNext())
            System.out.println("Thread " + threadNumber + ": " + new String(it.next().message()));
        System.out.println("Shutting down Thread: " + threadNumber);
    }
}
```
参考：[Consumer Group Example](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)

高级消费者API具有如下的优缺点：
- 优点
    1. API的使用很简单
    2. 不需要自己去管理消费的offset，API的内部会帮你管理
    3. 不需要涉及到分区、副本等比较底层的组件
    4. 加入了consumer group的概念了，不同group的消费程序消费的offset不会相互影响了
- 缺点
    1. 不能自行控制 offset（对于某些特殊需求来说）
    2. 不能细化控制如分区、副本、zk 等

### `1.0.0`版本消费者API
随着`API`的发展，Kafka通过`KafkaConsumer`将上面讲到的`低级消费者API`和`高级消费者API`统一起来，所以，我们在`0.9.x`版本后就可以通过`KafkaConsumer`的统一API来实现上面所有的消费场景了，我们来看几个例子：
1. 自动提交offset的场景
```java
Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("group.id", "test");
 // 设置为自动提交保存offset，默认就是true
 props.put("enable.auto.commit", "true");
 // 设置多长时间保存一次消费的offset，默认是5000ms
 // 所以说消费的offset并不是实时保存的，而是每隔一定的时间再保存一次
 props.put("auto.commit.interval.ms", "1000");
 props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
 consumer.subscribe(Arrays.asList("foo", "bar"));
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records)
         System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
 }
```
2. 手动控制消费的offset
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "test");
// 将自动提交保存offset的功能关闭掉
props.put("enable.auto.commit", "false");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("foo", "bar"));
final int minBatchSize = 200;
List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
while (true) {
 ConsumerRecords<String, String> records = consumer.poll(100);
 for (ConsumerRecord<String, String> record : records) {
     buffer.add(record);
 }
 // 当执行了一段业务逻辑后，达到了一定的条件后，再将消费的offset提交保存
 if (buffer.size() >= minBatchSize) {
     insertIntoDb(buffer);
     consumer.commitSync();  //手动同步提交保存offset
     buffer.clear();
 }
}
```
在上面的代码中，我们一批一批的消费消息，然后先将消息放在内存缓存中，当消费的消息的数量够了一个batch的大小后，则将这个batch中的所有的消息插入到数据库中。在这种场景下，如果我们设置自动保存offset的话，那么每一次`poll`后都会将消费的offset保存起来，当我们执行插入数据库的时候，可能失败了，但是这些消息都被认为已经消费了，这个有可能不符合我们的预期，插入失败的数据应该需要再次消费，所以我们设置手动控制提交offset，当数据都插入成功后再提交保存消费的offset

我们除了通过上面的`consumer.commitSync(); `将所有消费了的消息设置为已经消费，我们还可以指定某个`offset`来提交保存，如下，我们是消费完一个分区的消息，再提交保存`offset`
```java
try {
    while(running) {
        ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
        for (TopicPartition partition : records.partitions()) {
            List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
            for (ConsumerRecord<String, String> record : partitionRecords) {
                System.out.println(record.offset() + ": " + record.value());
             }
            long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
            // 提交保存消费当前分区消息的offset
            consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
        }
    }
} finally {
    consumer.close();
}
```

3. 手动指定分区消费消息
下面的代码就是指定消费两个分区的消息进行消费：
```java
String topic = "foo";
TopicPartition partition0 = new TopicPartition(topic, 0);
TopicPartition partition1 = new TopicPartition(topic, 1);
consumer.assign(Arrays.asList(partition0, partition1));
```

4. 手动控制消费者消费的位置
我们可以通过Consumer中[seek(TopicPartition, long)](https://kafka.apache.org/10/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#seek-org.apache.kafka.common.TopicPartition-long-) API来手动指定消费的位置

参考：[Kafka Consumer API](https://kafka.apache.org/10/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)

