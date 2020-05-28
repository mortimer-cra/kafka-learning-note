## 幂等性的作用

我们在 [Kafka的ack机制.md](Kafka的ack机制.md) 中说过，Kafka的`Producer`端有消息重发机制，那么可能会发生消息重复发送的情况，我们用下面的图来说明：

正常的情况如下： 

![image-20200526213722983](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526213928.png)

1. `Producer`端的`Sender`发送一个消息给`Broker Server`
2. `Broker Server`将消息追加到对应的分区的副本的数据文件中
3. 返回一个`ack`响应给客户端，表明消息已经成功发送

我们再来看一个消息重发的场景： 

![image-20200526215901231](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526215901.png)

1. `Producer`端的`Sender`发送一个消息给`Broker Server`
2. `Broker Server`将消息追加到对应的分区的副本的数据文件中
3. 返回一个`ack`响应给客户端，但是在返回`ack`的时候网络出问题了，客户端认为这次发送是失败的，但是这条消息已经存储在`Broker Server`中了
4. 客户端重新发送这条消息
5. `Broker Server`将消息再次将这个消息追加到对应的分区的副本的数据文件中
6. 返回一个`ack`响应给客户端，表明消息已经成功发送

从上面我们很容看出问题所在，就是消息重复的存储在`Broker Server`中了

Kafka为了解决消息重复存储的问题，在Producer端进行消息的幂等发送，在幂等发送中，Kafka在发送消息的时候，引入了两个字段：`ProduceId`和`Sequence Number`

`ProduceId`(简称`PID`)：当初始化一个新的`Producer`的时候，都会被分配一个唯一标识给这个新的`Producer`，也就是`ProducerId`，这个`ProducerId`对用户是不可见的

`Sequence Number`：`Producer`是发送一条消息给某个`topic`的某一个`partition`的，那么在一个`Producer`的每一个个`topic`的每一个`partition`都会对应着一个从`0`开始单调递增的`Sequence Number`(每次往这个分区发送消息的时候，就会将这个递增的`Sequence Number`放到这个消息中，作为这个消息在这个分区中的唯一标识)

`Broker Server`端会保存每一个消息对应的`ProduceId`和`Sequence Number`，对于同一个分区中接收到的每一条消息都会将这条消息的`Sequence Number`和上一个消息的`Sequence Number`对比，如果大，则会接收这个新消息，否则就会将这条消息丢弃掉，因为这条消息已经存在于这个分区中了，这样就可以解决消息的重复存储了

我们再来看下两个图，先看正常的情况：

![image-20200526215931951](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526215932.png)

1. `Producer`端的`Sender`发送一个消息给`Broker Server`，每一个消息都分配了`PID`和`Sequence Number`
2. `Broker Server`将消息追加到对应的分区的副本的数据文件中
3. 返回一个`ack`响应给客户端，表明消息已经成功发送

再来看一个异常的情况： 

![image-20200526220009352](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526220009.png)

1. `Producer`端的`Sender`发送一个消息给`Broker Server`，每一个消息都分配了`PID`和`Sequence Number`
2. `Broker Server`将消息追加到对应的分区的副本的数据文件中
3. 返回一个`ack`响应给客户端，但是在返回`ack`的时候网络出问题了，客户端认为这次发送是失败的，但是这条消息已经存储在`Broker Server`中了
4. 客户端重新发送这条消息，`PID`和`Sequence Number`还没变，所以和上一个消息的`Sequence Number`是一样大，表示是重复相同的消息，那么这次的消息会丢弃掉
5. 返回一个`ack`响应给客户端，说明消息是重复的

> 需要注意的是：Kafka的幂等只能保证单个Producer对于同一个<Topic, Partition>的幂等。不能保证同一个Producer一个topic不同的partition幂等。

## 幂等的使用

Kafka的幂等机制默认是关闭的，我们在初始化`Producer`的时候，需要将配置`enable.idempotence`设置为`true`，从而打开幂等机制，如下代码：

```java
package com.twq.kafka;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class IdempotentProducerTest {
    public static void main(String[] args) {
        Producer<String, String> producer = buildIdempotentProducer();
        producer.send(new ProducerRecord<>("my-topic", "key", "message"));
        producer.flush();
    }

    private static Producer<String, String> buildIdempotentProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        // 需要打开支持幂等的开关，默认是false
        // 此时就会默认把acks设置为all，所以不需要再设置acks属性了。
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
}
```