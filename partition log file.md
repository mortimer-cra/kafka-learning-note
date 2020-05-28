## 消息存储位置
每一个`topic`的每一个分区的数据都是存储在Kafka Broker Server所在的机器上的磁盘的，存储的目录是我们在`/home/hadoop-twq/bigdata/kafka_2.11-1.0.0/config/server.properties`中配置的以下的配置：
```shell
    ## 所有分区的数据存储的文件目录
    ## 这里可以配置多个文件目录，通过逗号隔开
    log.dirs=/home/hadoop-twq/bigdata/kafka-logs
```
一个分区的数据会对应着一个新的日志文件，我们称之为`partition log`

我们先创建一个`topic`，名称是`log-format`，分区数是`3`，每一个分区的备份数我们先设置为`1`，脚本如下：
```shell
    ## 在master机器上执行：
    cd ~/bigdata/kafka_2.11-1.0.0
    ## 创建一个名为log-format的topic，分区数是3，每一个分区的备份数我们先设置为`1`
    bin/kafka-topics.sh --create --zookeeper master:2181 --topic log-format --partitions 3 --replication-factor 1
```
创建完`log-format`后，我们看一眼三个Broker Server所在的机器上的文件目录`/home/hadoop-twq/bigdata/kafka-logs`的变化，如下：
![image-20200527105656502](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527105818.png)
从上面三张图我们可以看出：

1. 每一个Broker Server所在的机器上的Kafka存储数据的文件目录都会产生一个新的文件目录，名字为：`log-format-数字`，其中`log-format`表示`topic`的名称，而后面的数组则表示这个`topic`的分区数，例如，在master机器上，文件目录名称为`log-format-1`，则表示`log-format`这个`topic`的第`2`个分区数据存储的文件。所以`log-format`这个`topic`的第一个分区的数据存储在slav2机器上，第二个分区的数据存储在master机器上，而第三个分区的数据存储在slave1机器上
2. 我们再次验证了，每一个`topic`的所有分区都是均匀的分布在所有的Broker Server上

## partition Log
上面我们看了`3`个分区的数据存储的位置，接下来，我们详细看一下一个分区对应的存储文件目录下的存储结构，我们以`master`机器上的`log-format-1`为例，我们去看下这个文件目录有什么文件：
![image-20200527105853559](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527105853.png)

`log-format-1`下目前有`4`个文件，一个log数据文件、两个index索引文件以及一个checkpoint文件，在这篇文章中我们先不看checkpoint文件。我们来详细看其他的三个文件。

### 一个log数据文件
数据文件`00000000000000000000.log`就是用来存储消息的，我们先用下面的脚本来看一下`00000000000000000000.log`中的数据信息：
```shell
## 在master机器上执行：
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-		1/00000000000000000000.log --print-data-log 
```
输出的结果如下：
![image-20200527105933555](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527105933.png)
我们还没有给这个topic发送任何的消息，所以这个数据文件中肯定是没有消息数据呢。

我们尝试使用下面的`java`代码给这个`log-format`第`2`个分区发送数据(即在master机器上的分区)(**注意：我是直接运行下面的代码，运行了两次**)：
```java
package com.twq.kafka;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class LogFormatProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("batch.size", "10");

        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 1; i++) {
            producer.send(new ProducerRecord<String, String>("log-format", 1, //指定为第二个分区
                    Integer.toString(i), "this is for test partition log format"));
        }

        producer.close();
    }
}
```
运行两次上面的代码，给第二个分区发送了两条消息，然后我们再使用脚本来看下数据文件中的内容：
```shell
## 在master机器上执行：
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.log --print-data-log 
```
内容如下：
![image-20200527110010491](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527110010.png)
可以看出这个数据文件中确实有两条消息了，而且有两个偏移量`offset`了，上面的那些字段是啥意思，我们先不管，我们接着用下面的命令查看数据文件中两条消息的详细内容：

```shell
## 在master机器上执行：
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.log --print-data-log --deep-iteration
```
执行上面的命令后，输出是：
```shell
Starting offset: 0

offset: 0 position: 0 CreateTime: 1547003374605 isvalid: true keysize: 1 valuesize: 37 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 0 payload: this is for test partition log format

offset: 1 position: 106 CreateTime: 1547003869957 isvalid: true keysize: 1 valuesize: 37 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 0 payload: this is for test partition log format
```
上面就是数据文件中存储的真实数据了，我们来研究下每一条消息的每一个字段：
1. `offset`：就是每一条消息的偏移量，每一个分区的`offset`都是从`0`开始，然后往上递增的，所以上面第一条消息的`offset`是`0`，而第二条消息的`offset`是`1`
2. `position`：表示每一条消息存储在数据文件中的起始位置，第一条消息的起始位置当然是`0`，第二条消息的`position`是`106`，这个是因为第一条消息的大小是`105`字节，所以第二条消息的起始位置是`105 + 1`，至于为什么消息的长度是`105`，我们后面说
3. `CreateTime`：消息的创建时间戳
4. `isvalid`：表示消息的`crc`校验是否合法，在数据文件中是没有`isvalid`这个字段，这个字段是通过`crc`计算出来的，数据文件中存储的是`crc`这个字段，这个字段是表示一个消息的checksum的值，用于验证消息是否是合法的，如果消息被修改了，那么这个checksum的值就对不上，那么这个消息就不合法了，checksum值怎么计算可以不用管了
5. `keysize`：消息中的`key`的长度大小
6. `valuesize`：消息中的`value`的长度大小
7. `magic`：表示Kafka服务器协议版本号，用来做兼容，当前版本(`1.0.0`)的`magic`是`2`
8. `compresscodec`：消息被什么压缩算法压缩，这里两条消息都是`NONE`，说明消息没有被压缩
9. `producerId`和`sequence`是为了解决消息重复的问题，即幂等问题，详见： [Kafka的ack机制.md](Kafka的ack机制.md) 
10. `producerEpoch`和`isTransactional`两个字段是解决消息发送的事务问题，详见： [Kafka生产者事务详解.md](Kafka生产者事务详解.md) 
11. `headerKeys`：一条消息的头部信息中所有的`key`，这个header我们不经常使用

我们在谈论Producer原理的时候，我们的消息其实是一批一批的发给broker server的，每一批包含了多条消息，其实，在数据文件中消息的存储也是按照一批一批存储的，我们在上面是发送了`2`条消息，这`2`条消息是分开发的，所以是`2`个批次，每一个批次只有一个消息，我们现在运行下面的代码，一次性向第二个分区发送`3`条消息看看：
```java
    for (int i = 0; i < 3; i++) {
        producer.send(new ProducerRecord<String, String>("log-format", 1, //指定为第二个分区
                Integer.toString(i), "this is for test partition log format"));
    }
```
然后用下面的脚本查询数据文件的内容：
```shell
    ## 在master机器上执行：
    kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.log --print-data-log 
```
输出结果是：
```shell
Starting offset: 0
baseOffset: 0 lastOffset: 0 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 0 CreateTime: 1547003374605 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 505866327
baseOffset: 1 lastOffset: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 106 CreateTime: 1547003869957 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 812988848
baseOffset: 2 lastOffset: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 212 CreateTime: 1547014144070 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1668505285
baseOffset: 3 lastOffset: 3 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 318 CreateTime: 1547014144085 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 2729488342
baseOffset: 4 lastOffset: 4 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 424 CreateTime: 1547014144090 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1087373573
```
我们发现打印出`5`条消息来，因为我们确实是发送了`5`条消息，每一条消息的大小都会超过`10bit`，所以每一批中只能有一条消息了，我们尝试把`batch.size`设置的大一点，然后将`linger.ms`设置为`5000`，如下：
```java
    // 每一批消息的总大小，单位是字节
    props.put("batch.size", "1000"); 
    // 超过了这个时间没有发送消息的话，那也是一个新的批次消息了
    props.put("linger.ms", "500"); 
```
然后执行`Java`代码，再一次一次性的发送`3`消息，到现在为止已经发送了`2 + 3 + 3 = 8`条消息了，但是我们再一次利用上面的脚本查看数据文件得到的结果是：
```shell
Starting offset: 0
baseOffset: 0 lastOffset: 0 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 0 CreateTime: 1547003374605 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 505866327
baseOffset: 1 lastOffset: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 106 CreateTime: 1547003869957 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 812988848
baseOffset: 2 lastOffset: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 212 CreateTime: 1547014144070 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1668505285
baseOffset: 3 lastOffset: 3 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 318 CreateTime: 1547014144085 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 2729488342
baseOffset: 4 lastOffset: 4 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 424 CreateTime: 1547014144090 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1087373573
baseOffset: 5 lastOffset: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 530 CreateTime: 1547015227208 isvalid: true size: 196 magic: 2 compresscodec: NONE crc: 3913926735
```
只有`6`条记录，前面`5`条是我们之前的`5`条消息对应的批次，那么最后一条记录则是我们最后发送的`3`条消息的批次了，所以，我们的数据文件中其实是存储了`6`个批次的数据，前面`5`个批次每一个批次只有一条消息，而最后一个批次中则包含`3`条消息了，所以，我们可以得出结论：**数据文件中的存储是以batch为单位存储的，每一个batch可以有一到多条消息**

所以，现在我们可以来总结下数据文件中的存储格式了

#### 数据文件格式
1. 数据文件的消息是按照Batch来存储的，每一个Batch中包含1到多条消息，在Kafka中使用`RecordBatch`来表达，而每一条消息则是使用`Record`来表达
2. 每一个`RecordBatch`需要存储的字段结构如下：
```
baseOffset: int64                    //占8个字节，表示每这个batch第一个消息的offset
batchLength: int32                   //占4个字节，表示这个batch所有信息的长度大小
partitionLeaderEpoch: int32          //占4个字节，和事务相关
magic: int8 (current magic value is 2)      //占1个字节，表示Kafka服务器协议版本号
crc: int32                          //占4个字节，checksum的值，用于验证消息的有效性
attributes: int16                   //占2个字节，表示这个batch的一些属性信息
    bit 0~2:                        // 前3位用来标识压缩的算法，表示如下：
        0: no compression           // 000
        1: gzip                     // 001
        2: snappy                   // 010
        3: lz4                      // 100  
    bit 3: timestampType  //第4位表示时间类型，0表示CreateTime(消息创建时间)，1表示LogAppendTime(消息写到数据文件的时间)
    bit 4: isTransactional (0 means not transactional) // 第5位表示是否有事务
    bit 5: isControlBatch (0 means not a control batch) // 第6位表示是否属于控制批
    bit 6~15: unused                                    // 后面10位暂时没有用
lastOffsetDelta: int32  //占4个字节，等于最后一个消息的offset减去第一个消息的offset
firstTimestamp: int64   //占8个字节，表示第一个消息的创建时间
maxTimestamp: int64     //占8个字节，表示所有消息中创建时间最大的那么时间
producerId: int64                   //占8个字节，事务和幂等相关
producerEpoch: int16                //占2个字节，事务相关
baseSequence: int32                 //占4个字节，事务相关
numRecords: int32                   //占4个字节，表示这个batch包含多少条消息
records: [Record]                   // 这个batch中，所有的消息内容

// 除了records，batch元数据占用的空间是：61个字节
```
3. 每一个`Record`需要存储的字段结构如下：
```
length: varint      //压缩后的int，等于timestampDelta、offsetDelta、key、value所占大小的总和
attributes: int8                        //占1个字节，目前这个字段还没有用到
    bit 0~7: unused
timestampDelta: varlong      //压缩后的long，等于这条消息的timestampe减掉上面的firstTimestamp
offsetDelta: varint //压缩后的int，等于这条消息的offset减掉上面的baseOffset
keyLength: varint               //压缩后的int，key值所占空间的大小
key: byte[]                     //占可变的字节个数，表示key值的byte类型的值
valueLen: varint                //压缩后的int，value值所占空间的大小
value: byte[]                   //占可变的字节个数，表示value值的byte类型值
headersLength: varint           //压缩后的int，表示header的长度
Headers => [Header]             //占可变的字节个数，这个消息的头信息
```
`Record Header`的结构如下：
```
headerKeyLength: varint                                 //压缩后的int
headerKey: String                                       //占可变的字节个数
headerValueLength: varint                               //压缩后的int
Value: byte[]                                           //占可变的字节个数
```
> varint表示压缩之后的int值，其实就是用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数，这能减少用来表示数字的字节数。比如对于 int32 类型的数字，一般需要 4 个 byte 来表示。但是采用 Varint，对于很小的 int32 类型的数字(小于128的值)，则可以用 1 个 byte 来表示。

所以一个数据文件的结构如下图：
![image-20200527110248583](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527110248.png)

#### 每一个BatchRecord大小的计算
接下来，我们来看一下最后一个batch的大小：
```shell
## 这个是最后一个batch的元数据信息
baseOffset: 5 lastOffset: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 530 CreateTime: 1547015227208 isvalid: true size: 196 magic: 2 compresscodec: NONE crc: 3913926735
```
从上面我们可以看出，这个batch的size是`196`，那么这个`196`是怎么计算过来的呢，为了回答这个问题，我们还需要这个batch中的`3`条消息的详细信息，可以通过如下的脚本获得:
```shell
## 在master机器上执行：
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.log --print-data-log --deep-iteration
```
我们取`offset`从`5`到`7`这三条消息的详细信息，如下：
```shell
offset: 5 position: 530 CreateTime: 1547015227193 isvalid: true keysize: 1 valuesize: 37 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 0 payload: this is for test partition log format

offset: 6 position: 530 CreateTime: 1547015227208 isvalid: true keysize: 1 valuesize: 37 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: this is for test partition log format

offset: 7 position: 530 CreateTime: 1547015227208 isvalid: true keysize: 1 valuesize: 37 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 2 payload: this is for test partition log format
```
> 注意：上面这个batch中每一条消息的`position`是指这个batch record在文件中的起始的位置

从前面，我们得到一个batch元数据信息占用的空间：61个字节

3条消息占用的空间是：(1 + 1 + 1 + 1 + 1 + 1 + 1 + 37 + 1) * 3 = 135个字节，其中：

- length的长度为1字节 (因为length小于128)
- attributes的长度为1字节
- timestampDelta的长度为1字节
- offsetDelta的长度为1字节
- keyLength的长度为1字节
- key的长度为1字节
- valueLen的长度为1字节
- value的长度为37字节
- headersLength的长度为1字节

所以总大小为：61 + 135 = 196个字节，这个就是这个batch的长度为`196`的由来了。

#### 决定每一个数据文件的大小规则
随着消息往`log-format`这个`topic`的第二个分区发送消息越来越多，那么这个分区对应的数据文件`00000000000000000000.log`会越来越大了，我们总不可能让这个文件无限制的增大吧，否则的话，从这个文件中查询数据的性能会很慢，对于数据文件的大小，Kafka有如下两个规则来控制：
1. 默认当一个数据文件的大小大于1G(1 073 741 824字节)的时候，则会创建一个新的数据文件，这个二阈值大小由配置`segment.bytes`控制
2. 若数据文件大小没有达到配置`segment.bytes`的阈值，但是达到了`log.roll.ms`或是`log.roll.hours`设置的阈值，同样会创建一个新的数据文件。`log.roll.hours`默认是`168`小时

为了能看到`log-format`这个`topic`的第二个分区对应的数据文件可以被切分，我们将`segment.bytes`设置的小一点，我们设置`segment.bytes`为`5KB`。我们可以使用下面的命令来修改`log-format`对应的配置`segment.bytes`为`5120B`:
```shell
## 在任何一台机器上执行：
kafka-topics.sh -zookeeper master:2181,slave1:2181,slave2:2181 --alter --topic log-format --config segment.bytes=5120 

## 或者使用
bin/kafka-configs.sh --zookeeper master:2181,slave1:2181,slave2:2181 --entity-type topics --entity-name log-format --alter --add-config segment.bytes=5120
```
然后我们用下面的代码向`log-format`的第二个分区发送`20`条日志：
```shell
    for (int i = 0; i < 20; i++) {
        producer.send(new ProducerRecord<String, String>("log-format", 1, //指定为第二个分区
                Integer.toString(i), "this is for test partition log format"));
    }
```
去master的`/home/hadoop-twq/bigdata/kafka-logs/log-format-1`看还是一个数据文件，我们使用下面的命令可以查看现在数据文件的大小：
```shell
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.log --print-data-log
```
结果如下：
```
Starting offset: 0
baseOffset: 0 lastOffset: 0 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 0 CreateTime: 1547003374605 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 505866327
baseOffset: 1 lastOffset: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 106 CreateTime: 1547003869957 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 812988848
baseOffset: 2 lastOffset: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 212 CreateTime: 1547014144070 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1668505285
baseOffset: 3 lastOffset: 3 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 318 CreateTime: 1547014144085 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 2729488342
baseOffset: 4 lastOffset: 4 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 424 CreateTime: 1547014144090 isvalid: true size: 106 magic: 2 compresscodec: NONE crc: 1087373573
baseOffset: 5 lastOffset: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 530 CreateTime: 1547015227208 isvalid: true size: 196 magic: 2 compresscodec: NONE crc: 3913926735
baseOffset: 8 lastOffset: 20 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 726 CreateTime: 1547033458528 isvalid: true size: 649 magic: 2 compresscodec: NONE crc: 2541625495
baseOffset: 21 lastOffset: 27 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 1375 CreateTime: 1547033458535 isvalid: true size: 383 magic: 2 compresscodec: NONE crc: 1797934774
````
当前数据文件的大小等于最后一个batch的`position`再加上最后一个batch的`size`，即`1375 + 383 = 1758字节`，这个值小于`5KB`，所以数据文件不会被切分，我们继续向第二个分区发送`200条消息`：
```java
    for (int i = 0; i < 200; i++) {
        producer.send(new ProducerRecord<String, String>("log-format", 1, //指定为第二个分区
                Integer.toString(i), "this is for test partition log format"));
    }
```
再一次看下数据文件：
![image-20200527110551391](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527110623.png)
现在有`3`个数据文件了，名字分别是：`00000000000000000000.log`，`00000000000000000093.log`以及`00000000000000000184.log`，前面两个文件的大小各加上一个batch的数据肯定会超过`5KB即5120字节`的，所以被切分成3个数据文件了

从数据文件名，我们其实可以得出以下结论：
- `00000000000000000000.log`存储`offset`从`0`到`92`的消息
- `00000000000000000093.log`存储`offset`从`93`到`183`的消息
- `00000000000000000184.log`存储`offset`从`184`以及之后的消息

我们可以使用下面的命令，来验证`00000000000000000093.log`上存储的`offset`：
```shell
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000093.log --print-data-log
```
输出结果如下：
```
Starting offset: 93
baseOffset: 93 lastOffset: 105 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 0 CreateTime: 1547033949064 isvalid: true size: 659 magic: 2 compresscodec: NONE crc: 1330025272
baseOffset: 106 lastOffset: 118 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 659 CreateTime: 1547033949069 isvalid: true size: 659 magic: 2 compresscodec: NONE crc: 1396909489
baseOffset: 119 lastOffset: 131 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 1318 CreateTime: 1547033949073 isvalid: true size: 663 magic: 2 compresscodec: NONE crc: 4222932748
baseOffset: 132 lastOffset: 144 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 1981 CreateTime: 1547033949074 isvalid: true size: 672 magic: 2 compresscodec: NONE crc: 1271780422
baseOffset: 145 lastOffset: 157 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 2653 CreateTime: 1547033949074 isvalid: true size: 672 magic: 2 compresscodec: NONE crc: 3298177591
baseOffset: 158 lastOffset: 170 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 3325 CreateTime: 1547033949098 isvalid: true size: 672 magic: 2 compresscodec: NONE crc: 3272461508
baseOffset: 171 lastOffset: 183 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 1 isTransactional: false position: 3997 CreateTime: 1547033949098 isvalid: true size: 672 magic: 2 compresscodec: NONE crc: 1420576057
```
我们看第一个batch的`baseOffset=93`，最后一个batch的`lastOffset=183`，就可以知道`00000000000000000093.log`存储`offset`从`93`到`183`的消息

那么我们可以得出结论：数据文件的命名规则是，由数据文件的第一条消息偏移量，也就是基准偏移量(`baseOffset`)，左补`0`构成`20`位数字字符组成。
### 两个index索引文件
我们前面说过，一个数据文件(以`.log`结尾的)会对应着有两个索引文件，一个是以`.index`结尾(我们称之为偏移量索引文件)，还有一个是以`.timeindex`结尾的文件(我们称之为时间戳索引)，三个文件的命名规则是一样的，都是由数据文件的第一条消息偏移量，左补`0`构成`20`位数字字符组成。

接下来，我们分别来看一下这两个索引文件：

#### 偏移量索引文件
- 我们可以先通过下面的命令，看一下`00000000000000000000.index`文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.index --print-data-log
```
执行完后，得到下面的输出：
```
offset: 80 position: 4384
```
偏移量索引文件存储的每一条记录都是这个索引文件对应的数据文件中的某条信息的`offset`和`positition`，每一条记录我们称之为一个索引条目，比如上面的就是表示数据文件`00000000000000000000.log`中的`offset`为`80`的消息的信息，这条消息的在数据文件中起始存储位置是`第4384字节`

- 我们再通过下面的命令，看一下`00000000000000000093.index`这个索引文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000093.index --print-data-log
```
执行完后，发现没有输出，说明这个索引文件中没有存储数据

- 我们再通过下面的命令，看一下`00000000000000000184.index`这个索引文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000184.index --print-data-log
```
执行完后，得到下面的输出：
```
offset: 184 position: 0
```
可以看出，这个索引文件中存储的就是`offset`为`184`的消息的信息，这个消息在数据文件`00000000000000000184.log`中的存储位置为`第0个字节`

从上面，我们可以得出下面两点结论：
1. 每一个偏移量索引文件存储的是对应的数据文件中某条消息的`offset`和这条消息在这个数据文件中`position`
2. 不是所有的消息的`offset`和`position`都会存储在偏移量索引文件中，有一些索引文件可能还是为空

那么到底是哪一些消息会存储在偏移量索引文件中呢？这里有一个规则，每次写消息到数据文件时会检查是否要向索引文件写入索引条目，创建一个新的索引条目的条件为：距离前一次写索引后累计消息字节数大于配置`index.interval.bytes(默认是4096字节)`，第二个偏移量索引文件没什么没有索引条目数据就是因为第二个数据文件累积的消息字节数没有大于`4096`。

那么偏移量索引文件到底有什么用呢？我们先看下面的图，下面的图表示`log-format`这个`topic`第二个分区的偏移量索引文件和数据文件的关系：
![image-20200527110708464](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527110708.png)

> 注意：上面的每一个以.log结尾的文件，我们也可以称之为日志段

这个时候：
- 当消费者想从`offset`为`15`的消息开始消费的时候，那么是先去找这个消息对应的偏移量索引文件`00000000000000000000.index`(因为`0 < 15 < 93`)，然后发现这个索引文件中只有偏移量为`80`的索引，而`15 < 80`，这个时候就需要从数据文件`00000000000000000000.log`的第一条消息开始去查找，一直定位到`offset`为`15`的消息了
- 当消费者想从`offset`为`90`的消息开始消费的时候，那么是先去找这个消息对应的偏移量索引文件`00000000000000000000.index`(因为`0 < 90 < 93`)，发现这个`90`大于索引offset(80)，这个时候就需要从数据文件`00000000000000000000.log`的`offset`为`80`的消息开始去查找，一直定位到`offset`为`90`的消息了

所以，通过偏移量索引文件，我们就可以根据偏移量快速地定位到消息物理位置。首先根据指定的偏移量，通过二分查找，查询出该偏移量对应消息所在的数据文件和索引文件，然后在索引文件中通过二分查找，查找指小于等于指定偏移量的最大偏移量，最后从查找出的最大偏移量处开始顺序扫描数据文件，直至在数据文件中查询到偏移量与指定偏移量相等的消息

#### 时间戳索引文件
Kafka从`0.10.1.1`版本开始引入了一个基于时间戳的索引文件，即每一个日志端在物理上还对应着一个时间戳索引文件。
- 我们可以先通过下面的命令，看一下`00000000000000000000.timeindex`文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000000.timeindex --print-data-log
```
执行完后，得到下面的输出：
```
timestamp: 1547033949062 offset: 92
```
`timestamp`：表示该日志段目前为止最大时间戳
`offset`：表示插入当前新的索引条目的时候，当前消息的偏移量

- 我们再通过下面的命令，看一下`00000000000000000093.timeindex`这个索引文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000093.timeindex --print-data-log
```
执行完后，结果如下：
```
timestamp: 1547033949098 offset: 170
```

- 我们再通过下面的命令，看一下`00000000000000000184.timeindex`这个索引文件中有哪些数据信息：
```shell
# 在master机器上执行
kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/hadoop-twq/bigdata/kafka-logs/log-format-1/00000000000000000184.timeindex --print-data-log
```
执行上面的命令，输出我解释不了，感觉是Kafka的一个bug

不管怎么说，时间戳索引文件的功能和偏移量索引文件的功能是一样的，只不过这个时候不是根据偏移量来索引消息，而是根据时间戳来索引消息了

### 总结
我们上面的举例的`log-format`这个`topic`有`3`个分区，那么这个`topic`对应的消息的物理存储的结构如下：
![image-20200527111036334](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527111036.png)

1. 一个`topic`分成多个分区，分区是均匀的分布在所有的`broker server`上
2. 每一个分区在`broker server`所在的机器上的磁盘中都有一个对应的文件目录，用来存储这个分区的所有的消息信息，这个文件目录的名字是`topic_name + partition`。其实就是每一个分区对应着一个分区日志文件
3. 每一个分区对应的日志文件按照一定的规则被切分成多个日志段(LogSegment)
4. 每一个日志段包括了`3`个文件，一个是数据文件(`.log`结尾的)，一个是偏移量索引文件(`.index`结尾的)以及一个时间戳索引文件(`.timeindex`结尾的)
5. 数据文件就是存放消息的文件，消息是以`batch`为单位存放的，每一个`batch`中可以包含一到多条消息
6. 偏移量索引文件是用于根据`offset`来索引查找消息的
7. 时间戳索引文件是用于根据`timestamp`来索引查找消息的