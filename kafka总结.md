# kafka总结

[TOC]

## 什么是kafka

### 简介

kafka是一个分布式，分区的，多副本的，多订阅者的日志系统（分布式MQ系统），可以用于搜索日志，监控日志，访问日志等。

### 官方文档

https://kafka.apache.org/documentation/

### 有什么优点或特点

可靠性：分布式的，分区，复制和容错的。

可扩展性：kafka消息传递系统轻松缩放，无需停机。

耐用性：kafka使用分布式提交日志，这意味着消息会尽可能快速的保存在磁盘上，因此它是持久的。 

性能：kafka对于发布和订阅消息都具有高吞吐量。即使存储了许多TB的消息，他也爆出稳定的性能。 

kafka非常快：保证零停机和零数据丢失。

### 应用场景是什么

我们现在可以使用`Kafka`来实现如下的系统：

1. Messaging System(消息系统)
2. Storge System(存储系统，Kafka支持分布式数据存储，但是数据默认只存储`168h`，可以配置)
3. 连接器(数据导出导入)
4. Streaming Processing(流式处理)

## kafka整体架构是什么样的

### 总体架构图

![](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526200057.jpg)

### 各个角色的作用（术语）

#### Broker

一台kafka 服务器就是一个broker。一个集群由多个broker 组成。一个broker
可以容纳多个topic；

#### Topic

`Kafka`集群中的消息都是使用`topic`组织起来的，说白了，`topic`就是消息类别，也可以解释成主题，不同应用产生不同类别的消息，可以设置不同的主题

#### Partition

为了提高消息处理的并行度，一个`topic`的数据可以被分为多个`partition`(其实就是分区了)，每一个`partition`是一个有序、不变的消息序列，新的消息会不断的追加到这个消息序列中。在`partition`中每一条消息会按照时间顺序分配到一个单调递增的顺序编号，这个编号我们称之为消息的偏移量，即`offset`，如下图：

![image-20200526200814839](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526200814.png)

#### Producer

消息生产者，就是向kafka broker 发消息的客户端；

#### Consumer

消息消费者，向kafka broker 取消息的客户端；

#### Consumer Group

每一个Consumer属于一个特定的Consumer Group（可以为每个Consumer指定 groupName）

## kafka安装与部署

### 分布式安装kafka

 [分布式安装Kafka.md](分布式安装Kafka.md) 

### 常用的命令行操作

 [常用的命令行操作.md](常用的命令行操作.md) 

## kafka是怎么工作的

### kafka生产过程分析

#### 写入方式

producer 采用推（push）模式将消息发布到broker，每条消息都被追加（append）到分区（patition）中，属于顺序写磁盘（顺序写磁盘效率比随机写内存要高，保障kafka 吞吐率）

#### 分区

消息发送时都被发送到一个topic，其本质就是一个目录，而topic 是由一些PartitionLogs(分区日志)组成

**分区的原因**

- 方便在集群中扩展，每个Partition 可以通过调整以适应它所在的机器，而一个topic又可以有多个Partition 组成，因此整个集群就可以适应任意大小的数据了；
  
- 可以提高并发，因为可以以Partition 为单位读写了。

**分区的原则**

- 指定了patition，则直接使用；
- 未指定patition 但指定key，通过对key 的value 进行hash 出一个patition；
- patition 和key 都未指定，使用轮询选出一个patition。

#### 副本

同一个partition 可能会有多个replication （ 对应server.properties 配置中的default.replication.factor=N）。没有replication 的情况下，一旦broker 宕机，其上所有patition的数据都不可被消费，同时producer 也不能再将据存于其上的patition。引入replication之后，同一个partition 可能会有多个replication，而这时需要在这些replication 之间选出一个leader，producer 和consumer 只与这个leader 交互，其它replication 作为follower 从leader中复制数据。

#### 写入流程

![image-20200526203815781](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526203815.png)

**Producer中包含的组件有：**

- Sender：这个组件中包含一个NetworkClient，用于和Kafka中的Broker Server进行RPC通讯的，比如将数据发往某个Broker Server，或者从Kafka集群中查询某个topic的元数据信息
- Metadata：topic的元数据信息，比如这个topic包含哪些partition，每一个partition存在于哪一个Broker Server上，这部分信息是会缓存在Producer的内存中，但是到了一定的时候后，内存的元数据会失效，需要从Kafka集群中查询最新的元数据信息
- Partitioner：给消息分区的分区器
- Record Accumulator：用于将分区好的消息组织成一个batch，然后通过Sender，将这个batch的消息发往到对应的分区中

**一条消息发送的流程步骤如下(对应上图的步骤)：**

1. 根据`topic`向MetaData请求对应的topic的元数据
2. 如果topic的元数据不存在的话，则需要请求Sender向Kafka集群中查询该topic的元数据信息
3. 将查询到的topic的元数据更新到MetaData的内存中
4. 对消息进行分区，这个分区器是可以自定义的，如果没有自定义的话，则是使用Kafka默认的分区器，其分区的逻辑如下：
    1. 如果消息的`key`是空的，那么采用轮询的方式，即第一条消息发往`p0`，第二条消息则发往`p1`，第三条消息发往`p0`，第四条消息又发往`p1`..... 以此类推
    2. 如果消息的`key`不是空的话，则按照`key`的hash值与`topic`的分区数取模得到这条消息应该属于哪一个分区
5. 将分完区的消息追加到Record Accumulator相对应的分区数据上，当Record Accumulator达到以下两个条件之一的时候，则将接收到的消息组织成一个batch发往到Broker Server：
    1. 当Record Accumulator接收到的消息的速度大于向Kafka集群发送的速度，那么当Record Accumulator接收到的消息的大小达到了一个阈值(`batchSize`)
    2. 当Record Accumulator接收到的消息的速度小于向Kafka集群发送的速度，那么Record Accumulator持续接收消息，当达到一定的时间阈值(`lingerMs`)
6. 拿到Record Accumulator组织好的batch消息，并且拿到相应的分区存储的broker server信息
7. 通过组件Sender将batch消息发往到指定的broker server中

**可以得出一个结论**：消息的发送是一个异步的过程，即将消息分区后，然后追加到`Record Accumulator`就返回了

### Broker保存信息

#### 存储方式

物理上把topic 分成一个或多个patition（对应server.properties 中的num.partitions=3 配置），每个patition 物理上对应一个文件夹（该文件夹存储该patition 的所有消息和索引文件），如下：

![image-20200526204326404](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526204326.png)

#### 存储策略

无论消息是否被消费，kafka 都会保留所有消息。有两种策略可以删除旧数据：
1）基于时间：log.retention.hours=168
2）基于大小：log.retention.bytes=1073741824
需要注意的是，因为Kafka 读取特定消息的时间复杂度为O(1)，即与文件大小无关，所以这里删除过期文件与提高Kafka 性能无关。

#### zookeeper存储结构

![image-20200526204510649](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526204510.png)

注意：producer 不在zk 中注册，消费者在zk 中注册。

### kafka消费过程分析

kafka 提供了两套consumer API：高级Consumer API 和低级Consumer API。对于`Consumer API`，在版本`0.10.x`之前又分为两种：`高级消费API`和`低级消费API`。在`0.10.X`版本以及以后的版本中，这两种API被统一为`Consumer API`了，即不存在`高级消费API`和`低级消费API`的说法了

#### 高级API

优点

- 高级API 写起来简单
- 不需要自行去管理offset，系统通过zookeeper 自行管理。
- 不需要管理分区，副本等情况，系统自动管理。
- 消费者断线会自动根据上一次记录在zookeeper 中的offset 去接着获取数据（默认设置
  1 分钟更新一下zookeeper 中存的offset）
- 可以使用group 来区分对同一个topic 的不同程序访问分离开来（不同的group 记录不
  同的offset，这样不同程序读取同一个topic 才不会因为offset 互相影响）

缺点

- 不能自行控制offset（对于某些特殊需求来说）
- 不能细化控制如分区、副本、zk 等

#### 低级API

优点

- 能够让开发者自己控制offset，想从哪里读取就从哪里读取。
- 自行控制连接分区，对分区自定义进行负载均衡
- 对zookeeper 的依赖性降低（如：offset 不一定非要靠zk 存储，自行存储offset 即可，
  比如存在文件或者内存中）

缺点

- 太过复杂，需要自行控制offset，连接哪个分区，找到分区leader 等。

#### 消费者组

![image-20200526205103036](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200526205103.png)

消费者是以consumer group 消费者组的方式工作，由一个或者多个消费者组成一个组，共同消费一个topic。每个分区在同一时间只能由group 中的一个消费者读取，但是多个group可以同时消费这个partition。在图中，有一个由三个消费者组成的group，有一个消费者读取主题中的两个分区，另外两个分别读取一个分区。某个消费者读取某个分区，也可以叫做某个消费者是某个分区的拥有者。在这种情况下，消费者可以通过水平扩展的方式同时读取大量的消息。另外，如果一个消费者失败了，那么其他的group 成员会自动负载均衡读取之前失败的消费者读取的分区。

#### 消费方式

Kafka的Consumer是消费Broker Server的消息，采用是`pull`机制，即消费者是从Broker Server端拉取的消息，然后进行消费。Kafka之所以选择使用`pull`机制，是因为考虑到如下的两点优点：
1. 当一个消费者消费消息落后的时候，当它想赶上消息进度的时候，都是很好控制的
2. 更加容易控制批量的消费消息，如果是`push`机制的话，要么就是来了一条消息就消费一条消息，要么在Broker端累积一定的消息后，然后`push`给消费者，但是呢，消费者不一定立马可以处理这个`batch`消息，所以呢，如果消费者掌握主动权的话，就可以很容易的控制批量消费了

## kafka API

见 [kafka常用api.md](kafka常用api.md) 

### 概述

### Producer API

### Consumer API

- 0.9.x版本消费者API

	- 低级消费者API
	- 高级消费者API

- 1.0.0版本消费者API

### Streams API简述

Kafka Streams。Apache Kafka 开源项目的一个组成部分。是一个功能强大，易于使用的
库。用于在Kafka 上构建高可分布式、拓展性，容错的应用程序。

### Java开发Producer和Consumer

#### Producer的代码

```java
package com.twq.streaming.kafka.example;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class SimpleProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        // Kafka Broker Server的监听地址，多个可以使用逗号分隔开
        props.put("bootstrap.servers", "master:9092,slave1:9092,slave2:9092");
        
        // ack机制相关配置
        props.put("acks", "all");
        
        // 重发机制
        props.put("retries", 2);
        
        // 批量发送的相关参数
        props.put("buffer.memory", 33554432);
        props.put("compression.type", "snappy");
        props.put("batch.size", 10);
        props.put("linger.ms", 1);
        
        // key-value序列化相关配置
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        

        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 100; i++) {
            producer.send(new ProducerRecord<String, String>("test-group",
                    Integer.toString(i), Integer.toString(i)));
        }

        producer.close();
    }
}
```

我们现在来详细解释下上面提到的我们不怎么熟悉的几个参数：

1. ack机制相关的两个配置参数：

```java
    // acks可以的取值为[0, 1, -1或者all]， -1和all的功能是一样的
    props.put("acks", "all"); 
```

ack机制的讲解请详细参考： [Kafka的ack机制.md](Kafka的ack机制.md) 

1. 重发机制

```java
    props.put("retries", 2);
```

当`Producer`发送消息的时候，如果消息发送失败的话，则会尝试重发这个消息，这个配置默认的值是`0`，即默认是禁用重发机制

1. 批量发送的相关四个配置参数

```java
    props.put("buffer.memory", 33554432);
    props.put("compression.type", "snappy");
    props.put("batch.size", 10);
    props.put("linger.ms", 1);
```

我们知道，`Producer`发送消息的时候，是先对消息进行分区，然后将分区之后的消息发送到`Record Accumulator`这个内存缓存中。然后将消息组织成`batch`再发送，下面是对上面四个参数配置的说明：

- `Record Accumulator`这个内存缓存的大小就是用配置`buffer.memory`来控制，这个配置的大小是`33554432字节(即32M)`，这个参数建议配置成比`Producer`进程堆内存大小小一点最好
- 每一条消息发往`Record Accumulator`后，可以通过配置参数`compression.type`来决定是否需要对消息进行压缩，压缩算法的类型有：none、gzip、snappy以及lz4，默认是none(即对消息不压缩)
- 消息是在`Record Accumulator`中组织成`batch`再发送到`Broker Server`中的，那么这个`batch`的大小由配置`batch.size(单位是字节)`来控制了，这个参数如果设置太小的话就会使得发送消息的次数变多，也可能降低了吞吐量；当然，如果设置太大的话可能会浪费内存，也会导致消息发送的延迟。这个值默认是`16384字节`
- 通常来说只有在消息产生速度大于发送速度的时候才会将消息组织成`batch`，然后再一起发送，那如果消息产生的速度小于发送的速度的话，当然是来一条消息就发送一条消息了，这样的话消息的发送就没有延迟了，但是来一条消息就发送一条消息的话，会导致请求数太多了，就是需要不断的网络传输，会有性能损耗，如果我们设置一个时间段，当消息来了的时候，先不发送消息，让消息等待一段时间，在这段时间内可能会等到其他的消息，然后再将这段时间内的消息当成一个`batch`再发送，这样可以减少发送请求的次数，这个时间段我们可以通过`linger.ms(单位是毫秒)`配置来设置，这个配置默认的值是`0`，表示来一条消息发送一条消息，就不进行等待了

1. key-value序列化相关的两个配置参数：

```java
    props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
```

`key.serializer`和`value.serializer`的功能分别是将消息的`key`和`value`序列化成字节(因为网络传输传输的是字节)

Kafka支持常用的序列化器有：

```java
// 字符串类型
org.apache.kafka.common.serialization.StringSerializer
org.apache.kafka.common.serialization.JsonSerializer
// byte类型的数组
org.apache.kafka.common.serialization.ByteArraySerializer
org.apache.kafka.common.serialization.ByteBufferSerializer
// 基本类型
org.apache.kafka.common.serialization.DoubleSerializer
org.apache.kafka.common.serialization.FloatSerializer
org.apache.kafka.common.serialization.IntegerSerializer
org.apache.kafka.common.serialization.LongSerializer
org.apache.kafka.common.serialization.ShortSerializer
```

如何选择消息的格式，请参考： [Kafka消息数据类型的选择.md](Kafka消息数据类型的选择.md) 

#### Consumer的代码

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
        
        // key-value反序列化的配置
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(Arrays.asList("test-group"));

        while (true) {
            // 采用的是pull模式来消费消息
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

有两点需要着重说明下：

1. key-value反序列化的两个配置

```java
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
```

`key.deserializer`和`value.deserializer`的功能分别是将字节类型消息的`key`和`value`反序列化成对应的类型(因为从网络传输传输过来的字节)

Kafka支持常用的反序列化器有：

```java
// 字符串类型
org.apache.kafka.common.serialization.StringDeserializer
org.apache.kafka.common.serialization.JsonDeserializer
// byte类型的数组
org.apache.kafka.common.serialization.ByteArrayDeserializer
org.apache.kafka.common.serialization.ByteBufferDeserializer
// 基本类型
org.apache.kafka.common.serialization.DoubleDeserializer
org.apache.kafka.common.serialization.FloatDeserializer
org.apache.kafka.common.serialization.IntegerDeserializer
org.apache.kafka.common.serialization.LongDeserializer
org.apache.kafka.common.serialization.StringDeserializer
```

如何选择消息的格式，请参考： [Kafka消息数据类型的选择.md](Kafka消息数据类型的选择.md) 

### 总结

1. 以前的低级消费者API其实就是消费者可以自己控制消费的分区、副本以及消费的offset，相对来说灵活点，但是灵活的同时会使得代码更加的复杂，消费者需要自己管理分区、副本以及对应的消费offset，还需要处理leader副本变更的情况
2. 以前的高级消费者API使用起来相对就比较简单了，因为高级API就是关注消费的消息内容，而不去关注和管理消费的分区、副本以及offset了
3. 从`0.9.x`版本后，Kafka使用新的消费者API--`KafkaConsumer`来统一了低级API和高级API了，在`KafkaConsumer`中既能实现低级API实现的功能，也能实现高级API实现的功能

## kafka原理与机制解读

### partition log存储机制

 [partition log file.md](partition log file.md) 

### CAP理论以及kafka当中的ISR机制

#### CAP理论概述

分布式系统（distributed system）正变得越来越重要，大型网站几乎都是分布式的。分布式系统的最大难点，就是各个节点的状态如何同步。

为了解决各个节点之间的状态同步问题，在1998年，由加州大学的计算机科学家 Eric Brewer 提出分布式系统的三个指标，分别是

Consistency：一致性

Availability：可用性

Partition tolerance：分区容错性

Eric Brewer 说，这三个指标不可能同时做到。这个结论就叫做 CAP 定理

具体理解参考 [CAP 定理的含义](http://www.ruanyifeng.com/blog/2018/07/cap.html)

#### kafka当中的CAP应用

kafka是一个分布式的消息队列系统，既然是一个分布式的系统，那么就一定满足CAP定律，那么在kafka当中是如何遵循CAP定律的呢？kafka满足CAP定律当中的哪两个呢？

kafka满足的是CAP定律当中的CA，其中Partition tolerance通过的是一定的机制尽量的保证分区容错性。

其中C表示的是数据一致性。A表示数据可用性。

kafka首先将数据写入到不同的分区里面去，每个分区又可能有好多个副本，数据首先写入到leader分区里面去，读写的操作都是与leader分区进行通信，保证了数据的一致性原则，也就是满足了Consistency原则。然后kafka通过分区副本机制，来保证了kafka当中数据的可用性。但是也存在另外一个问题，就是副本分区当中的数据与leader当中的数据存在差别的问题如何解决，这个就是Partition tolerance的问题。

kafka为了解决Partition tolerance的问题，使用了ISR的同步策略，来尽最大可能减少Partition tolerance的问题

#### ISR机制

每个leader会维护一个ISR（a set of in-sync replicas，基本同步）列表

ISR列表主要的作用就是决定哪些副本分区是可用的，也就是说可以将leader分区里面的数据同步到副本分区里面去，决定一个副本分区是否可用的条件有两个

replica.lag.time.max.ms=10000   副本分区与主分区心跳时间延迟

replica.lag.max.messages=4000  副本分区与主分区消息同步最大差

具体验证见[Kafka的ISR机制.md](Kafka的ISR机制.md) 

### kafka的ack机制

 [Kafka的ack机制.md](Kafka的ack机制.md) 

## kafka的监控与运维

### 脚本kafka-consumer-groups.sh的使用

 [查看Consumer消费的情况.md](查看Consumer消费的情况.md) 

### 开源工具

参考：https://www.kafka-eagle.org/

