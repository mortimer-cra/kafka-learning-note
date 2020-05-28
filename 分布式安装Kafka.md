### 下载上传解压
- 下载地址：[kafka_2.11-1.0.0.tgz](http://archive.apache.org/dist/kafka/1.0.0/kafka_2.11-1.0.0.tgz)。
    1. 这个安装文件中的`2.11`是指使用的`Scala`版本是`2.11.X`的版本，`Kafka`主要是使用`Scala`语言实现的
    2. `1.0.0`是指`Kafka`的版本，我们就使用这个版本
- 将安装包`kafka_2.11-1.0.0.tgz`上传到`master`机器上的`~/bigdata`下，然后使用下面的命令解压：
```shell
    tar -xzf kafka_2.11-1.0.0.tgz
```

### 在master上修改配置
```shell
    cd ~/bigdata/kafka_2.11-1.0.0/config
    vi server.properties
    ## 修改两个参数：
        log.dirs=/home/hadoop-twq/bigdata/kafka-logs-new
        zookeeper.connect=master:2181
    ## 创建一个目录：
    mkdir ~/bigdata/kafka-logs-new
```
### 将master上的安装包scp到slave1和slave2
```shell
    scp -r ~/bigdata/kafka_2.11-1.0.0 hadoop-twq@slave1:~/bigdata/
    scp -r ~/bigdata/kafka_2.11-1.0.0 hadoop-twq@slave2:~/bigdata/

    scp -r ~/bigdata/kafka-logs-new hadoop-twq@slave1:~/bigdata/
    scp -r ~/bigdata/kafka-logs-new hadoop-twq@slave2:~/bigdata/
```

### 修改slave1和slave2上的配置
```shell
    cd ~/bigdata/kafka_2.11-1.0.0/config
    vi server.properties
    
    修改一个参数：
        slave1上为：broker.id=1
        slave2上为：broker.id=2
```

### 分别在master、slave1和slave2上启动broker server
```shell
    cd ~/bigdata/kafka_2.11-1.0.0
    mkdir logs
    
    ## 分别在master、slave1和slave2上执行下面的命令，启动broker server
    nohup bin/kafka-server-start.sh config/server.properties >~/bigdata/kafka_2.11-1.0.0/logs/server.log 2>&1 &
```
然后`3`个服务器上通过`jps`命令都可以看到`Kafka Broker Server`的进程名`Kafka`，如下：
![kafka9](BA86A44ECB4F430887E9DB0400983052)
![kafka10](2D595E61908C413197656A37740AD495)
![kafka11](A646631A65F545339A0620BB8609698B)

### 创建并查看topic
```shell
    ## 在master机器上执行：
    cd ~/bigdata/kafka_2.11-1.0.0
    ## 创建一个名为test-1的topic
    bin/kafka-topics.sh --create --zookeeper master:2181 --replication-factor 1 --partitions 1 --topic test-1
    
    ## 通过下面的命令可以查看指定的topic
    bin/kafka-topics.sh --list --zookeeper master:2181
```

### 启动producer发送消息以及启动consumer消费消息
```shell
    ## 在任何一台机器上执行下面的命令，启动一个往test-1这个topic上发送消息的Producer
    cd ~/bigdata/kafka_2.11-1.0.0
    bin/kafka-console-producer.sh --broker-list master:9092 --topic test-1
    
    ## 在任何一台机器上执行下面的命令，启动一个消费test-1这个topic上的消息
    cd ~/bigdata/kafka_2.11-1.0.0
    bin/kafka-console-consumer.sh --bootstrap-server master:9092 --topic test-1 --from-beginning
```
消息发送的图片：
![kafka7](F19D50C5D0DB4BBB8C3131BC902F5C0B)
消息接收的图片(可以忽略WARN的日志)：
![kafka8](ACAF9E8A859541549B76F8B93DF48A43)
到此说明你的Kafka集群正常安装并且正常启动

### 关闭Kafka Broker Server
```shell
    ## 分别在三台虚拟上执行下面的命令关闭Broker Server
    bin/kafka-server-stop.sh
```