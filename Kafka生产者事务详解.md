## Kafka事务
说到事务，我们都知道传统数据库，比如Oracle和Mysql，都是支持事务的，在一个事务中的所有的数据库操作，要么全部成功，要么全部失败，先看一下下面的伪代码：
```
    begin transaction
        update table1;
        delete table2 where ...;
        update table2;
    end transaction
```
如果是`JDBC`代码的话，上面的`transaction`是使用数据库的`commit`和`rollback`来实现的，在`transaction`之间的所有数据库操作就是一个原子操作，要么全部成功，要么全部失败

Kafka是在`0.11.0.0`版本中引入的事务，Kafka的事务和上面说的事务类似，只不过上面在事务中操作的是数据库，而Kafka中在事务中操作的是Kafka，Kafka引入事务的目的有俩：
1. 在前面的讲的幂等性只能保证同一个`topic`的同一个`parition`中的消息的幂等，对于跨`partition`的就无能为力了，如果一个`producer`向多个`partition`发送消息时怎么样保证要么都成功，要么都失败呢，只能依靠事务了
2. 在流式处理中，我们经常会碰到这样一个场景：先从Kafka中消费消息，然后对消息进行处理，处理完之后再将结果以消息的形式写到Kafka中。这种场景我们称之为`consumer-transform-producer`模式，在这种场景下，由于程序的失败，consumer很容易重复消费数据，这样就导致producer发送了重复的数据了，那么可以使用事务来解决这种场景的消息重复问题，将`consumer-transform-producer`中的所有与Kafka的操作都放在同一个事务中，作为原子操作。

从上面我们可以看出，不管是幂等性还是`事务`，其实都是为了使得Kafka的消息传递语义达到`Exactly Once`

从上面我们还能看出，Kafka的事务是指：**在同一个事务中一系列的生产者生产消息和消费者消费消息的操作是原子操作**

我们来看两个场景的代码：
### 保证多个消息发送的操作的原子性
```java
public static void main(String[] args) {
        producersInTransaction();
}

/**
 *  在一个事务只有生产消息操作
 */
public static void producersInTransaction() {
    Producer producer = buildTransactionProducer();
    // 1.初始化事务,对于一个生产者,只能执行一次初始化事务操作
    producer.initTransactions();
    // 2.开启事务
    producer.beginTransaction();
    try {
        // 3.kafka消息发送的操作集合
        // 3.1 do业务逻辑

        // 3.2 向相同的topic的不同的partition发送消息
        producer.send(new ProducerRecord<String, String>("my-topic", "data-1"));

        producer.send(new ProducerRecord<String, String>("my-topic", "data-2"));
        
        // 3.3 还可以向不同的topic发送消息
        producer.send(new ProducerRecord<String, String>("other-topic", "data-3"));
        
        // 3.4 do其他业务逻辑。

        // 4.事务提交，说明上面Kafka消息发送的操作全部成功
        producer.commitTransaction();
    } catch (Exception e) {
        // 5.放弃事务， 其实就是回滚操作。
        // 说明上面Kafka消息发送的操作全部失败
        producer.abortTransaction();
    }
}

private static Producer<String, String> buildTransactionProducer() {
    Properties props = new Properties();
    props.put("bootstrap.servers", "master:9092");
    // 必须设置事务id
    props.put("transactional.id", "first-transactional");
    // 此时enable.idempotence必须被设置为true
    props.put("enable.idempotence", true);
    
    props.put("retries", 3);
    props.put("linger.ms", 1);
    props.put("buffer.memory", 33554432);
    props.put("key.serializer",
            "org.apache.kafka.common.serialization.StringSerializer");
    props.put("value.serializer",
            "org.apache.kafka.common.serialization.StringSerializer");

    Producer<String, String> producer = new KafkaProducer<String, String>(props);

    return producer;
}
```
这里需要注意的点：
1. 在初始化`Producer`的时候必须设置`transactional.id`，可以保证多个相同`transactional.id`的`Producer`开始新的事务之前先完成旧的事务，英文解释如下：
```
The TransactionalId to use for transactional delivery. This enables reliability semantics which span multiple producer sessions since it allows the client to guarantee that transactions using the same TransactionalId have been completed prior to starting any new transactions
```
2. enable.idempotence必须被设置为true
### 保证`consumer-transform-producer`所有消息操作为原子操作
```java
public static void main(String[] args) {
    consumeTransformProduce();
}

/**
 * 在一个事务内,即有生产消息又有消费消息
 */
public static void consumeTransformProduce() {
    // 1.构建生产者
    Producer producer = buildTransactionProducer();
    // 2.初始化事务,对于一个生产者,只能执行一次初始化事务操作
    producer.initTransactions();
    // 3.构建消费者和订阅主题
    Consumer consumer = buildConsumer();
    consumer.subscribe(Arrays.asList("my-topic"));
    while (true) {
        // 4.开启事务
        producer.beginTransaction();
        // 5.1 消费消息
        ConsumerRecords<String, String> records = consumer.poll(500);
        try {
            // 5.2 do业务逻辑;
            // 这个是用于保存需要提交保存的consumer消费的offset
            Map<TopicPartition, OffsetAndMetadata> commits = new HashMap<>();
            for (ConsumerRecord<String, String> record : records) {
                // 5.2.1 读取消息,并处理消息。
                // 这里作为示例，只是简单的将消息打印出来而已
                System.out.printf("offset = %d, key = %s, value = %s\n",
                        record.offset(), record.key(), record.value());

                // 5.2.2 记录提交的消费的偏移量
                commits.put(new TopicPartition(record.topic(), record.partition()),
                        new OffsetAndMetadata(record.offset()));

                // 6.生产新的消息。比如外卖订单状态的消息,如果订单成功,则需要发送跟商家结转消息或者派送员的提成消息
                producer.send(new ProducerRecord<String, String>("other-topic", "data2"));
            }
            // 7.提交偏移量，
            // 其实这里本质上是将消费者消费的offset信息发送到名为__consumer-offset中的topic中
            producer.sendOffsetsToTransaction(commits, "group1");

            // 8.事务提交
            producer.commitTransaction();
        } catch (Exception e) {
            // 7.放弃事务。回滚
            producer.abortTransaction();
        }
    }
}

/**
 * 需要:
 * 1、关闭自动提交 enable.auto.commit
 * 2、isolation.level为read_committed
 */
public static Consumer buildConsumer() {
    Properties props = new Properties();
    props.put("bootstrap.servers", "master:9092");
    props.put("group.id", "group1");
    
    // 设置隔离级别
    // 如果设置成read_committed的话，那么consumer.poll(500)的时候只会读取事务已经提交了的消息
    // 如果设置成read_uncommitted，那么consumer.poll(500)则会消费所有的消息，即使事务回滚了的消息也会消费
    // 默认的行为是read_uncommitted
    props.put("isolation.level","read_committed");
    // 关闭自动提交
    // 必须关闭自动提交保存消费者消费的offset，因为是用事务中的API来提交了
    props.put("enable.auto.commit", "false");

    props.put("key.deserializer",
            "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer",
            "org.apache.kafka.common.serialization.StringDeserializer");

    KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);

    return consumer;

}
```
这里需要注意的点：
1. `consumer-transform-producer`在 [Kafka Stream](https://kafka.apache.org/documentation/streams/)中 用的比较多，Kafka Stream使用这种场景的事务可以达到`Exactly Once`的语义
2. 需要消费者的自动提交`offset`的属性值设置为false,并且不能再手动的调用执行`consumer.commitSync`或者`consumer.commitAsyc`

### 总结
1. Kafka和事务相关的API都是在`Producer`中
2. 事务属性实现前提是幂等性，即在配置事务属性transaction id时，必须还得配置幂等性；但是幂等性是可以独立使用的，不需要依赖事务属性。


## 参考
1. [Kafka生产者事务和幂等](http://www.heartthinkdo.com/?p=2040)
2. [Exactly Once语义与事务机制原理](http://www.jasongj.com/kafka/transaction/)
