## ISR机制
我们说一个`topic`可以分成多个`partition`，每一个`partition`的消息数据都存储在某个`broker server`的磁盘文件中，但是，当某个`partition`所在的`broker server`挂掉了，那么这个`partition`的消息就不能对外服务了，为了解决这个问题，Kafka设计了`ISR`机制，其实基本的思想就是将一个`partition`的数据备份到多个`broker server`上(和HDFS的`Block`备份的思想是一样的)，这样就可以提高每一个`patition`的高可用性。

我们先用下面的命令，创建一个`topic`，名称为`kafka-isr`，我们设置这个`topic`为`3`个分区，每一个分区`3`个备份
```shell
    ## 在master机器上执行：
    cd ~/bigdata/kafka_2.11-1.0.0
    ## 创建一个名为kafka-isr的topic，分区数是3，每一个分区的备份数设置为3
    bin/kafka-topics.sh --create --zookeeper master:2181 --topic kafka-isr --partitions 3 --replication-factor 3
```
执行完上面的命令后，我在在每一台机器上执行下面的命令：
```shell
 ll /home/hadoop-twq/bigdata/kafka-logs
```
截图如下：
![image-20200527113052258](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527113149.png)

可以看出，`3`个分区的数据都存储在`3`个`broker server`上了，也就是说每一个分区都有`3`个备份的数据了，他们都是均匀的分布在`3`个`broker server`上。

> 上图中每一个文件目录存储的数据就是每一个分区的每一个备份的数据，里面存储数据的结构和我们在 [partition log file.md](partition log file.md) 中讲解的是一模一样的

我们接下来再使用下面的命令查看下`kafka-isr`这个`topic`的信息：
```shell
bin/kafka-topics.sh --describe --zookeeper master:2181 --topic kafka-isr
```
输出结果如下：
![image-20200527113224557](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527114011.png)

从上图我们可以看出：
1. 这个`topic`有`3`个分区，每个分区有`3`个备份(也可以称之为副本)
2. `Replicas`：是指每一个分区的副本数据存储的`broker server id`的列表(我们在`server.properties`中有配置`broker.id`的)，比如分区`0`的副本数据存储在`broker.id`等于`[2, 0, 1]`三个`broker server`中
3. `Leader`：是指每一个分区的`3`个副本的`Leader`备份。一个分区中的若干个副本中，肯定会有`1`个`Leader副本`，`0`个会多个`Follower副本`；`Leader副本`处理分区的所有的读写请求并维护自身以及`Follower副本`的状态信息，`Follower副本`作为消费者从`Leader副本`拉取消息进行同步；当`Leader副本`失效时，通过分区`Leader`选举器从`Isr`列表中选出一个副本作为新的`Leader副本`
4. `Isr`：是 `in sync replicas`的缩写，其实，说白了就是一个分区存活的`备份`列表，那么如果一个`备份`符合下面的两个要求，则表名它是存活的：
    1. 这个`备份`必须和`zookeeper`保持通讯(通过`Zookeeper`的心跳机制)
    2. 这个`备份`必须正在同步`Leader备份`的消息，并且不能落后很多
5. 当一个`备份`不符合上面的两个条件，即认为这个`备份`是死的状态，那么就会从这个`ISR`列表中移除掉

> 一定要注意：上图中的`Leader: 2`这个2是指broker.id=2，其实就是slave2上的broker server。Replicas以及Isr后面的数字列表也是指broker.id列表

我们再看下下面的图，可能会更清晰点：
![image-20200527114033426](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527114033.png)

我们现在尝试将`slave1`上的`broker server`杀掉，命令如下：
```shell
    # 在slave1上执行
    cd /home/hadoop-twq/bigdata/kafka_2.11-1.0.0
    bin/kafka-server-stop.sh
```

我们接下来再使用下面的命令查看下`kafka-isr`这个`topic`的信息：
```shell
bin/kafka-topics.sh --describe --zookeeper master:2181 --topic kafka-isr
```
截图如下：
![image-20200527114055412](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527114055.png)

我们可以看出以下的改变点：
1. 所有分区的`Isr`的列表中现在只有两个`broker server id`了，因为`slave1`上的`broker server`挂了，即`broker.id=1`的节点被移除了
2. 第三个分区的`Leader副本`变成了`2`了


我们再尝试将`slave1`上的`broker server`启动起来，命令如下：
```shell
nohup ~/bigdata/kafka_2.11-1.0.0/bin/kafka-server-start.sh ~/bigdata/kafka_2.11-1.0.0/config/server.properties >~/bigdata/kafka_2.11-1.0.0/logs/server.log 2>&1 &
```
我们接下来再使用下面的命令查看下`kafka-isr`这个`topic`的信息：
```shell
bin/kafka-topics.sh --describe --zookeeper master:2181 --topic kafka-isr
```
截图如下：
![image-20200527114109979](https://raw.githubusercontent.com/mortimer-cra/mypic/master/img/20200527114110.png)
我们可以看出：所有分区的`Isr`又恢复了`3`个`broker.id`的列表了

## 如果所有的副本所在的机器都挂了，怎么办？
如果一个分区至少有一个副本存活的话，那么Kafka可以保证这个分区的消息记录不会丢，但是当整个分区的所有副本所在的`broker server`都挂了的话，那么，Kafka就不一定能保证消息不丢失了。

当你真的遇到这种情况后，对于你来说，最重要的是要知道接下来会发生什么事情，一般的话有下面两种行为中的一种行为可能会发生：
1. 等待在这个分区中的`ISR`列表中的副本恢复，然后将这个恢复的副本作为`Leader副本`(当然是希望所有的数据仍然都在)
2. 选择第一个副本(不一定是在`ISR`列表中的副本)进行恢复，并作为`Leader副本`

上面两种行为其实是在`可用性`和`一致性`之间做的一个取舍平衡，等待`ISR`列表中的副本恢复作为Leader，可能可以保证数据的一致性(数据不会丢)，但是需要时间，在等待的过程中，那么这个分区就不可用了；如果直接选择第一个副本进行恢复并作为Leader，不需要时间，这个分区立马可以工作有用，但是数据可能丢失了，因为它不一定在`ISR`列表中。Kafka默认的是选择第二种策略了，当然我们可以通过配置`unclean.leader.election.enable=true`来禁用第二种策略了