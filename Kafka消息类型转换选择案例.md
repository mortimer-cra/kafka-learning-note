```
080449201DAA T 00000195700000103000N0000000000000004CT1000100710071007 
080449201DAA T 00000131000000107000N0000000000000005CT10031007 
080449201DAA T 00000066600000089600N0000000000000006CT10041005 
080449201DAA T 00000180200000105100N0000000000000007CT100310051009 
080449201DAA T 00000132200000089700N0000000000000008CT100410051005 
080449201DAA T 00000093500000089400N0000000000000009CT10031007 
080449201DAA T 00000075100000105400N0000000000000010CT100410081006 
080449201DAA T 00000031300000088700N0000000000000011CT1004100810081007 
```
上面的数据是一个股票行业的数据，每一行数据是一个定长的字符串数据，我们在这篇文章中只有其中的四个字段值：
- 0-8个字符表示时间(date)
- 9-15个字符表示一些标识(symbol)
- 16-25个字符表示价格(price)
- 26-38个字符表示容量(volume)

我们可以利用下面的`Producer`代码，将上面的数据发送到Kafka中的名为`raw-data`的`topic`中：
```java
Properties props = new Properties();
props.put("bootstrap.servers", "master:9092");
props.put("batch.size", "10");
// 发送消息的key是Integer类型
props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
// 发送消息的key是String类型
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<Integer, String> producer = new KafkaProducer<>(props);

producer.send(new ProducerRecord<Integer, String>("raw-data",
            1, "080449201DAA T 00000195700000103000N0000000000000004CT1000100710071007"));

producer.send(new ProducerRecord<Integer, String>("raw-data",
        2, "080449201DAA T 00000131000000107000N0000000000000005CT10031007"));

producer.close();
```
分别启动3个Consumer来消费`raw-data`中的消息，而且分别将消息解析为：JsonObject、Java Bean以及byte类型的数组。如下：
### **JSONObject**
```java
package com.twq.kafka.dataType;

import com.alibaba.fastjson.JSONObject;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Arrays;
import java.util.Properties;

public class RawData2JsonConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "rawdata-group");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<Integer, String> consumer = new KafkaConsumer<Integer, String>(props);
        consumer.subscribe(Arrays.asList("raw-data")); // 消费raw-data这个topic中的消息

        while (true) {
            ConsumerRecords<Integer, String> records = consumer.poll(100);
            for (ConsumerRecord<Integer, String> record : records) {
                Integer key = record.key();
                String rawData = record.value();

                // 这个时候，我们的业务想访问股票原始数据中的字段
                // 我们可以将每一个消息转换成JsonObject
                JSONObject jsonObject = new JSONObject();
                jsonObject.put("date", rawData.substring(0,9));
                jsonObject.put("symbol", rawData.substring(9,16));
                jsonObject.put("price", rawData.substring(16,26));
                jsonObject.put("volume", rawData.substring(26,39));

                // 然后我们就可以通过 jsonObject 来访问相应的字段，比如
                System.out.println(jsonObject.get("symbol")); // 访问 symbol字段

                // 我们还可以将数据以json的格式发往给另一个名为json-data的topic
                buildProducer().send(new ProducerRecord<Integer, String>("json-data",
                        key, jsonObject.toJSONString()));
            }
        }
    }
}
```
这种方法有以下三个缺陷：
1. 没有数据验证(影响数据的使用的方便性，说白了就是访问字段不是很方便)
2. 会产生大量的JsonObject对象(影响性能)
3. 有很多的字符串切割的操作(影响性能)
### **Java Bean**
```java
package com.twq.kafka.dataType;

public class TickBean {
    private String date;
    private String symbol;
    private String price;
    private String volume;

    public String getDate() {
        return date;
    }

    public void setDate(String date) {
        this.date = date;
    }

    public String getSymbol() {
        return symbol;
    }

    public void setSymbol(String symbol) {
        this.symbol = symbol;
    }

    public String getPrice() {
        return price;
    }

    public void setPrice(String price) {
        this.price = price;
    }

    public String getVolume() {
        return volume;
    }

    public void setVolume(String volume) {
        this.volume = volume;
    }
}
```
```java
package com.twq.kafka.dataType;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectOutput;
import java.io.ObjectOutputStream;
import java.util.Arrays;
import java.util.Properties;

public class RawData2BeanConsumer {
    public static void main(String[] args) throws IOException {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "rawdata-group2");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<Integer, String> consumer = new KafkaConsumer<Integer, String>(props);
        consumer.subscribe(Arrays.asList("raw-data")); // 消费raw-data这个topic中的消息

        while (true) {
            ConsumerRecords<Integer, String> records = consumer.poll(100);
            for (ConsumerRecord<Integer, String> record : records) {
                Integer key = record.key();
                String rawData = record.value();

                // 这个时候，我们的业务想访问股票原始数据中的字段
                // 我们可以将每一个消息转换成Tick类型的Java Bean
                TickBean tick = new TickBean();
                tick.setDate(rawData.substring(0,9));
                tick.setSymbol(rawData.substring(9,16));
                tick.setPrice(rawData.substring(16,26));
                tick.setVolume(rawData.substring(26,39));

                // 然后我们就可以通过 tick 来访问相应的字段，比如
                System.out.println(tick.getSymbol()); // 访问 symbol字段
            }
        }
    }
}

```
转成Java Bean后，使得访问字段的值变得很方便了，方便性和性能比JsonObject的都有了提高，但是还是会存在如下的两个缺陷：
1. Java对象的属性对内存的使用效率不高(因为对象的属性可能会被放在内存效率比较低的位置上)
2. 还是存在大量的字符串切割的操作，影响性能
### **byte类型的数组**
```java
package com.twq.kafka.dataType;

public class Tick {
    private byte[] data;

    public Tick(byte[] data) {
        this.data = data;
    }

    public byte[] getData() {
        return this.data;
    }

    public String getDate() {
        return new String(data, 0, 9);
    }

    public String getSymbol() {
        return new String(data, 10, 15);
    }

    public String getPrice() {
        return new String(data, 16, 25);
    }

    public String getVolume() {
        return new String(data, 26, 39);
    }
}
```

```java
package com.twq.kafka.dataType;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;

import java.io.IOException;
import java.util.Arrays;
import java.util.Properties;

public class RawData2ByteConsumer {
    public static void main(String[] args) throws IOException {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "rawdata-group2");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer");

        KafkaConsumer<Integer, byte[]> consumer = new KafkaConsumer<Integer, byte[]>(props);
        consumer.subscribe(Arrays.asList("raw-data")); // 消费raw-data这个topic中的消息

        while (true) {
            ConsumerRecords<Integer, byte[]> records = consumer.poll(100);
            for (ConsumerRecord<Integer, byte[]> record : records) {
                Integer key = record.key();
                byte[] rawData = record.value();

                // 这个时候，我们的业务想访问股票原始数据中的字段
                // 我们可以将每一个消息转换成Tick类型的Java Bean(这个Bean中存储的是byte类型的数组)
                Tick tick = new Tick(rawData);

                // 然后我们就可以通过 tick 来访问相应的字段，比如
                System.out.println(tick.getSymbol()); // 访问 symbol字段
            }
        }
    }
}
```
转成存储byte类型的数组是一个效率比较高的方式，首先byte类型的数组本来在内存中就是一块连续的内存块，其访问效率很好，而且我们是直接通过数组的索引来访问字段的值，从而消除了字符串切割带来的性能损耗，但是这种方式也是会有一个问题，就是：

1. 当消费的消息的数据格式不对(长度不对，缺少或者到了字段)的话，那么这里的数据解析就会错误了