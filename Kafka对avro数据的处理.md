如果我们使用`avro`作为消息的类型的话，我们需要先定义`avro`的schema文件，假设我们定义了一个`StockQuotation.avsc`的文件，如下：
```json
{
    "namespace": "com.twq.kafka.dataType.avro",
    "type": "record",
    "name": "StockQuotation",
    "fields": [
        {"name": "stockCode", "type": "string"},
        {"name": "stockName", "type": "string"},
        {"name": "tradeTime", "type": "long"},
        {"name": "preClosePrice", "type": "float"},
        {"name": "openPrice", "type": "float"},
        {"name": "currentPrice", "type": "float"},
        {"name": "highPrice", "type": "float"},
        {"name": "lowPrice", "type": "float"}
    ]
}
```

然后使用`avro`的maven插件将上面的文件翻译成代码，生成的Java类的名称就是`StockQuotation`

我们现在用`Producer`将上面的`avro`数据发送给Kafka中名为`avro-tt`的`topic`，脚本和代码如下：
```shell
    ## 先用下面的脚本创建这个topic
    cd ~/bigdata/kafka_2.11-1.0.0
    bin/kafka-topics.sh --create --zookeeper master:2181 --replication-factor 1 --partitions 1 --topic avro-tt
```

1. 先自定义avro类型的序列化器
```java
package com.twq.kafka.dataType.avro;

import org.apache.avro.io.BinaryEncoder;
import org.apache.avro.io.DatumWriter;
import org.apache.avro.io.EncoderFactory;
import org.apache.avro.specific.SpecificDatumWriter;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.Map;

/**
 *  自定义avro类型的序列化器
 */
public class AvroSerializer implements Serializer<StockQuotation> {
    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> arg0, boolean arg1) {}

    @Override
    public byte[] serialize(String topic, StockQuotation data) {
        if(data == null) {
            return null;
        }
        // 将 StockQuotation 类型的数据序列化成 byte 类型的数组
        DatumWriter<StockQuotation> writer = new SpecificDatumWriter<>(data.getSchema());
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        BinaryEncoder encoder = EncoderFactory.get().directBinaryEncoder(out, null);
        try {
            writer.write(data, encoder);
        }catch (IOException e) {
            throw new SerializationException(e.getMessage());
        }
        return out.toByteArray();
    }
}
```
2. 自定义avro类型的反序列化器
```java
package com.twq.kafka.dataType.avro;

import org.apache.avro.io.BinaryDecoder;
import org.apache.avro.io.DatumReader;
import org.apache.avro.io.DecoderFactory;
import org.apache.avro.specific.SpecificDatumReader;
import org.apache.kafka.common.serialization.Deserializer;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.util.Map;

/**
 *  自定义avro反序列化器
 */
public class AvroDeserializer implements Deserializer<StockQuotation> {

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> arg0, boolean arg1) {}

    @Override
    public StockQuotation deserialize(String topic, byte[] data) {
        if(data == null) {
            return null;
        }
        StockQuotation stockQuotation = new StockQuotation();
        // 将字节类型的数组反序列化成 StockQuotation 对象
        ByteArrayInputStream in = new ByteArrayInputStream(data);
        DatumReader<StockQuotation> userDatumReader = new SpecificDatumReader<>(stockQuotation.getSchema());
        BinaryDecoder decoder = DecoderFactory.get().directBinaryDecoder(in, null);
        try {
            stockQuotation = userDatumReader.read(null, decoder);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return stockQuotation;
    }
}
```
3. 消息发送代码：
```java
package com.twq.kafka.dataType.avro;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class AvroDataProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("batch.size", "10");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        // 使用自定义的avro的序列化器
        props.put("value.serializer", "com.twq.kafka.dataType.avro.AvroSerializer");

        Producer<Integer, StockQuotation> producer = new KafkaProducer<>(props);

        StockQuotation.Builder builder = StockQuotation.newBuilder();
        builder.setStockCode("0009");
        builder.setStockName("aqiyi");
        builder.setTradeTime(167234222L);
        builder.setPreClosePrice(10.5F);
        builder.setOpenPrice(11.5F);
        builder.setCurrentPrice(12.3F);
        builder.setHighPrice(13.7F);
        builder.setLowPrice(10.11F);

        StockQuotation stockQuotation = builder.build();

        producer.send(new ProducerRecord<Integer, StockQuotation>("avro-tt", 1, stockQuotation));

        producer.close();
    }
}
```
4. 消息消费代码
```java
package com.twq.kafka.dataType.avro;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class AvroDataConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "avrodata-group");
        // 从最早的消息开始消费
        props.put("auto.offset.reset", "earliest");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        // 使用自定义的avro的反序列化机制
        props.put("value.deserializer", "com.twq.kafka.dataType.avro.AvroDeserializer");

        KafkaConsumer<Integer, StockQuotation> consumer = new KafkaConsumer<Integer, StockQuotation>(props);
        consumer.subscribe(Arrays.asList("avro-tt"));

        while (true) {
            ConsumerRecords<Integer, StockQuotation> records = consumer.poll(100);
            for (ConsumerRecord<Integer, StockQuotation> record : records) {
                StockQuotation stockQuotation = record.value();

                System.out.println(stockQuotation.getStockCode());
                System.out.println(stockQuotation.getStockName());
            }
        }
    }
}
```

参考书本：[Kafka入门与实践.牟大恩](https://pan.baidu.com/s/1NjMlIJKbhWYRH1yO-IxdDQ)