Kafka默认支持消息的数据类型有：String、 byte[]、Double、Float、Integer、Long、Short，因为Kafka的消息都是以字节的方式传递、存储的，所以每一种数据类型都会对应着序列化器和反序列化器，用来在数据类型和字节之间进行相互转换。

比如，我们经常使用的String类型，我们在发送String类型的消息时需要指定String类型的序列化器，如下代码：
```java
    props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    
    Producer<String, String> producer = new KafkaProducer<>(props);
    producer.send(new ProducerRecord<String, String>("test-group",
                    "keyStr", "this must be a string value"));
```

在消费数据的时候，我们想可以通过指定String对应的反序列化器将字节类型的消息反序列化成String字符串，如下图：
```java
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    
    KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
    consumer.subscribe(Arrays.asList("test-group"));
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            String key = record.key();
            String value = record.value();
        }
    }
```

如果我们需要传递的消息非常的简单，用这种key-value的消息就可以表达的话，那么我们的消息就选择String这个数据类型了，但是有很多的场景，单单的使用String类型是不够的。除了这种简单的String类型的消息之外，我们还可以使用`Json`、`avro`等数据类型来表达我们需要传递的消息：
- 怎么样传递以及处理`Json`类型的消息，我们可以参考： [Kafka对Json消息的处理.md](Kafka对Json消息的处理.md) 
- 怎么样传递以及处理 `avro` 类型的消息，可以参考： [Kafka对avro数据的处理.md](Kafka对avro数据的处理.md) 

那么接下来，我们来看一下怎么样选择合适消息的数据类型：
1. 对于简单的消息，我们当然就直接使用`String`类型的消息了
2. `Json`本质上也是String类型的，只不过是半结构化的字符串
    1. `Json`的优点：
        1. 灵活，数据格式可以不是固定的，还可以支持嵌套的数据结构
        2. `Json`在数据转换方面的工作量比较小，而且基本所有的语言都支持`Json`数据的解析
    2. `Json`的缺点：
        1. `Json`的每条记录中的每一个字段都是`key-value`类型的数据，所以会存在`key`数据冗余，当数据量特别大的时候，就会有大量的数据冗余了，降低了数据存储的效率。如果在Kafka中存储这个`Json`数据的话，存储也是要考虑的重要因素
        2. `Json`的操作，存在大量的字符串截取操作，也非常的影响性能
3. `avro`类型的数据则解决了`Json`类型数据的一些问题，可以提高存储效率，也没有字符串截取操作，相对来说性能会更优，但是呢`avro`多了一到数据转换的工序

所以，当数据格式不固定的话，则使用`Json`类型的格式，当数据格式是固定的话，那建议还是使用`avro`类型的格式。

另外，在消费者端将消息解析转换成什么类型的性能和效率更高，请参考 [这个案例](Kafka消息类型转换选择案例.md) 

