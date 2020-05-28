`Json`的样例数据如下：
```json
{"payload":{"username":"john"},"metadata":{"eventName":"Login","sessionId":"089acf50-00bd-47c9-8e49-dc800c1daf50","username":"john","hasSent":null,"createDate":1511186145471}}
```

对于Json类型的消息，可以使用Kafka内置的String序列化器和反序列化器，也可以使用自定义的，接下来我们来看下两种情况：

## 使用Kafka内置的String序列化器和反序列化器
我们现在用`Producer`将上面的`Json`数据发送给Kafka中名为`json-tt`的`topic`，脚本和代码如下：
```shell
    ## 先用下面的脚本创建这个topic
    cd ~/bigdata/kafka_2.11-1.0.0
    bin/kafka-topics.sh --create --zookeeper master:2181 --replication-factor 1 --partitions 1 --topic json-tt
```

Java发送代码如下：
```java
package com.twq.kafka.dataType.json;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class JsonDataProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("batch.size", "10");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        // 使用Kafka内置的String序列化器
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        Producer<Integer, String> producer = new KafkaProducer<>(props);

        String json = "{\"payload\":{\"username\":\"john\"},\"metadata\":{\"eventName\":\"Login\",\"sessionId\":\"089acf50-00bd-47c9-8e49-dc800c1daf50\",\"username\":\"john\",\"hasSent\":null,\"createDate\":1511186145471}";
        producer.send(new ProducerRecord<Integer, String>("json-tt",
                    1, json));

        producer.close();
    }
}
```
我们使用下面的消费代码来消费`json-tt`这个`topic`中的消息：
```java
package com.twq.kafka.dataType.json;

import com.alibaba.fastjson.JSONObject;
import com.twq.kafka.dataType.TickBean;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class JsonDataConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "jsondata-group");
        // 从最早的消息开始消费
        props.put("auto.offset.reset", "earliest");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        // 使用Kafka内置的String反序列化器
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<Integer, String> consumer = new KafkaConsumer<Integer, String>(props);
        consumer.subscribe(Arrays.asList("json-tt"));

        while (true) {
            ConsumerRecords<Integer, String> records = consumer.poll(100);
            for (ConsumerRecord<Integer, String> record : records) {
                String jsonData = record.value(); // 这里读出来的是Json字符串数据
                // 这个时候，我们的业务想访json数据中的字段
                // 我们可以将每一个json字符串类型的消息转换成JSONObject
                JSONObject jsonObject =  (JSONObject)JSONObject.parse(jsonData);
                // 然后我们就可以通过 jsonObject 来访问相应的字段，比如
                System.out.println(jsonObject.getJSONObject("payload")); // 访问 payload字段
                System.out.println(jsonObject.getJSONObject("metadata")); // 访问 metadata 字段
                
                // 我们也可以将 jsonObject 转成普通的Java Bean，这个当然是跟着业务走了
            }
        }
    }
}
```
上面的代码依赖阿里巴巴的`fastjson`，maven依赖如下：
```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.54</version>
</dependency>
```

## 自定义的序列化器和反序列化器
我们除了可以使用阿里巴巴的`fastjson`外，我们还可以使用`jackson`的`json`解析包，下面的代码就是依赖`jackson`的解析包，我们不用导入任何额外的依赖，因为kafka是依赖`jackson`的

我们先自定义一个`Json`的序列化器和反序列化器：

1. 自定义Json的序列化器：
```java
package com.twq.kafka.dataType.json;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;

import java.util.Map;

// 自定义序列化器，就是实现org.apache.kafka.common.serialization.Serializer这个接口即可
public class JsonSerializer implements Serializer<JsonNode> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void configure(Map<String, ?> map, boolean b) {

    }
    @Override
    public byte[] serialize(String s, JsonNode jsonNode) {
        if (jsonNode == null)
            return null;

        try {
            // 将 JsonNode 类型的对象序列化成字节类型的数组
            return objectMapper.writeValueAsBytes(jsonNode);
        } catch (Exception e) {
            throw new SerializationException("Error serializing JSON message", e);
        }
    }
    @Override
    public void close() {
        
    }
}
```
2. 自定义Json的反序列化器：
```java
package com.twq.kafka.dataType.json;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Deserializer;

import java.util.Map;

// 自定义反序列化就是实现org.apache.kafka.common.serialization.Deserializer接口即可
public class JsonDeserializer implements Deserializer<JsonNode> {
    private final ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public void configure(Map<String, ?> map, boolean b) {

    }
    @Override
    public JsonNode deserialize(String s, byte[] bytes) {
        if (bytes == null)
            return null;

        JsonNode data;
        try {
            // 将字节类型数组反序列化成 JsonNode
            data = objectMapper.readTree(bytes);
        } catch (Exception e) {
            throw new SerializationException(e);
        }

        return data;
    }
    @Override
    public void close() {

    }
}
```
3. 发送`JsonNode`类型的消息
```java
package com.twq.kafka.dataType.json;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.io.IOException;
import java.util.Properties;

public class JsonNodeProducer {
    public static void main(String[] args) throws IOException {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("batch.size", "10");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        // 指定value的序列化器是我们自定义的序列化器
        props.put("value.serializer", "com.twq.kafka.dataType.json.JsonSerializer");

        Producer<Integer, JsonNode> producer = new KafkaProducer<>(props);

        String json = "{\"payload\":{\"username\":\"john\"},\"metadata\":{\"eventName\":\"Login\",\"sessionId\":\"089acf50-00bd-47c9-8e49-dc800c1daf50\",\"username\":\"john\",\"hasSent\":null,\"createDate\":1511186145471}}";
        ObjectMapper objectMapper = new ObjectMapper();
        // 将JSON转成JsonNode
        JsonNode jsonNode = objectMapper.readTree(json);
        producer.send(new ProducerRecord<Integer, JsonNode>("json-tt", 1, jsonNode));

        producer.close();
    }
}
```
4. 消费`JsonNode`类型消息的代码
```java
package com.twq.kafka.dataType.json;

import com.fasterxml.jackson.databind.JsonNode;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class JsonNodeConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("group.id", "jsondata-group");
        // 从最早的消息开始消费
        props.put("auto.offset.reset", "earliest");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        // 使用自定义的Json反序列化器
        props.put("value.deserializer", "com.twq.kafka.dataType.json.JsonDeserializer");

        KafkaConsumer<Integer, JsonNode> consumer = new KafkaConsumer<Integer, JsonNode>(props);
        consumer.subscribe(Arrays.asList("json-tt"));

        while (true) {
            ConsumerRecords<Integer, JsonNode> records = consumer.poll(100);
            for (ConsumerRecord<Integer, JsonNode> record : records) {
                JsonNode jsonNode = record.value();

                // 然后我们就可以通过 jsonNode 来访问相应的字段，比如
                System.out.println(jsonNode.get("payload").toString()); // 访问 payload字段
                System.out.println(jsonNode.get("metadata").toString()); // 访问 metadata 字段

                // 我们也可以将 jsonNode 转成普通的Java Bean，这个当然是跟着业务走了
            }
        }
    }
}
```