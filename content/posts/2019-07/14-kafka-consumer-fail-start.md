---
title: "kafka consumer无法正常启动原因调查"
date: 2019-07-14T17:10:46+08:00
draft: false
---

# 背景
生产环境三个服务分别消费kafka(version 1.1.0)不同topic里的日志分析结果，将其写入HBase，其中消费kafka使用的是consumer的subscribe API。服务稳定运行了几个月，某次因为HBase升级将其临时关闭，在重新启动过程中，诡异的问题发生了。

# 现象
三个服务中有一个可以正常启动，另外两个在启动过程中似乎卡在某个地方block住。打开info级别的日志，发现如下可疑信息
> 2018-12-28 16:58:22 WARN pool-1-thread-2 org.apache.kafka.clients.NetworkClient - [Consumer clientId=consumer-2, groupId=defy-rtlog-receiver-srcproxy] Connection to node -2 could not be established. Broker may not be available.

调整日志级别为debug，重新观察观察日志，发现服务不停打印如下信息
> Coordinator discovery failed, refreshing metadata

# 分析问题
从日志来看，服务中kafka consumer一直死循环寻找coordinator. 但coordinator是什么鬼？

话说源码之前了无秘密，翻下源码，可以从注释中可以看到GroupCoordinator角色的描述如下

>  GroupCoordinator handles general group membership and offset management.
>
> Each Kafka server instantiates a coordinator which is responsible for a set of
 groups. Groups are assigned to coordinators based on their group names.


即GroupCoordinator负责管理group的offset，每一个broker启动一个GroupCoordinator，负责管理一些consumer groups。

既然每一个broker都有一个GroupCoordinator，那么如何判断一个group归哪个GroupCoordinator来管呢？

这里涉及到kafka内部一个特殊的topic `__consumer_offsets`，其用于存储group的消费情况，默认有50个partition。

某一group消费情况要存在哪一个partition，通过以下方式得到
```
  def partitionFor(groupId: String): Int = Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
```
计算得到的partition的leader所在的Broker即为该group对应的GroupCoordinator。

有了上面的原理铺垫，再来看日志描述，排查方向变得清晰起来。用命令kafka-topic.sh看下topic `__consumer_offsets`
```
JMX_PORT=9844 ./kafka-topics.sh --zookeeper jjh715:2181/kafka-jjh-cluster1 --describe --topic __consumer_offsets
# output
Topic:__consumer_offsets        PartitionCount:50       ReplicationFactor:1     Configs:segment.bytes=104857600,cleanup.policy=compact,compression.type=producer
        Topic: __consumer_offsets       Partition: 0    Leader: 5       Replicas: 5     Isr: 5
        Topic: __consumer_offsets       Partition: 1    Leader: 6       Replicas: 6     Isr: 6
        Topic: __consumer_offsets       Partition: 2    Leader: 7       Replicas: 7     Isr: 7
        Topic: __consumer_offsets       Partition: 3    Leader: 8       Replicas: 8     Isr: 8
        Topic: __consumer_offsets       Partition: 4    Leader: 9       Replicas: 9     Isr: 9
        Topic: __consumer_offsets       Partition: 5    Leader: 0       Replicas: 0     Isr: 0
        Topic: __consumer_offsets       Partition: 6    Leader: 1       Replicas: 1     Isr: 1
        Topic: __consumer_offsets       Partition: 7    Leader: 2       Replicas: 2     Isr: 2
        Topic: __consumer_offsets       Partition: 8    Leader: -1      Replicas: 3     Isr: 3
        Topic: __consumer_offsets       Partition: 9    Leader: 4       Replicas: 4     Isr: 4
        Topic: __consumer_offsets       Partition: 10   Leader: 5       Replicas: 5     Isr: 5
        Topic: __consumer_offsets       Partition: 11   Leader: 6       Replicas: 6     Isr: 6
        Topic: __consumer_offsets       Partition: 12   Leader: 7       Replicas: 7     Isr: 7
        Topic: __consumer_offsets       Partition: 13   Leader: 8       Replicas: 8     Isr: 8
        Topic: __consumer_offsets       Partition: 14   Leader: 9       Replicas: 9     Isr: 9
        Topic: __consumer_offsets       Partition: 15   Leader: 0       Replicas: 0     Isr: 0
        Topic: __consumer_offsets       Partition: 16   Leader: 1       Replicas: 1     Isr: 1
        Topic: __consumer_offsets       Partition: 17   Leader: 2       Replicas: 2     Isr: 2
        Topic: __consumer_offsets       Partition: 18   Leader: -1      Replicas: 3     Isr: 3
        Topic: __consumer_offsets       Partition: 19   Leader: 4       Replicas: 4     Isr: 4
        Topic: __consumer_offsets       Partition: 20   Leader: 5       Replicas: 5     Isr: 5
        Topic: __consumer_offsets       Partition: 21   Leader: 6       Replicas: 6     Isr: 6
        Topic: __consumer_offsets       Partition: 22   Leader: 7       Replicas: 7     Isr: 7
        Topic: __consumer_offsets       Partition: 23   Leader: 8       Replicas: 8     Isr: 8
        Topic: __consumer_offsets       Partition: 24   Leader: 9       Replicas: 9     Isr: 9
        Topic: __consumer_offsets       Partition: 25   Leader: 0       Replicas: 0     Isr: 0
        Topic: __consumer_offsets       Partition: 26   Leader: 1       Replicas: 1     Isr: 1
        Topic: __consumer_offsets       Partition: 27   Leader: 2       Replicas: 2     Isr: 2
        Topic: __consumer_offsets       Partition: 28   Leader: -1      Replicas: 3     Isr: 3
        Topic: __consumer_offsets       Partition: 29   Leader: 4       Replicas: 4     Isr: 4
        Topic: __consumer_offsets       Partition: 30   Leader: 5       Replicas: 5     Isr: 5
        Topic: __consumer_offsets       Partition: 31   Leader: 6       Replicas: 6     Isr: 6
        Topic: __consumer_offsets       Partition: 32   Leader: 7       Replicas: 7     Isr: 7
        Topic: __consumer_offsets       Partition: 33   Leader: 8       Replicas: 8     Isr: 8
        Topic: __consumer_offsets       Partition: 34   Leader: 9       Replicas: 9     Isr: 9
        Topic: __consumer_offsets       Partition: 35   Leader: 0       Replicas: 0     Isr: 0
        Topic: __consumer_offsets       Partition: 36   Leader: 1       Replicas: 1     Isr: 1
        Topic: __consumer_offsets       Partition: 37   Leader: 2       Replicas: 2     Isr: 2
        Topic: __consumer_offsets       Partition: 38   Leader: -1      Replicas: 3     Isr: 3
        Topic: __consumer_offsets       Partition: 39   Leader: 4       Replicas: 4     Isr: 4
        Topic: __consumer_offsets       Partition: 40   Leader: 5       Replicas: 5     Isr: 5
        Topic: __consumer_offsets       Partition: 41   Leader: 6       Replicas: 6     Isr: 6
        Topic: __consumer_offsets       Partition: 42   Leader: 7       Replicas: 7     Isr: 7
        Topic: __consumer_offsets       Partition: 43   Leader: 8       Replicas: 8     Isr: 8
        Topic: __consumer_offsets       Partition: 44   Leader: 9       Replicas: 9     Isr: 9
        Topic: __consumer_offsets       Partition: 45   Leader: 0       Replicas: 0     Isr: 0
        Topic: __consumer_offsets       Partition: 46   Leader: 1       Replicas: 1     Isr: 1
        Topic: __consumer_offsets       Partition: 47   Leader: 2       Replicas: 2     Isr: 2
        Topic: __consumer_offsets       Partition: 48   Leader: -1      Replicas: 3     Isr: 3
        Topic: __consumer_offsets       Partition: 49   Leader: 4       Replicas: 4     Isr: 4
```
仔细观察输出结果如下，发现部分partition（8，18，28，38，48）的leader为-1，replicas为3, 同时发现该topic的ReplicationFactor为1
![kafka-topic-output]()

leader为-1表明leader所在的broker目前not available，replicas为3表明之前该broker为3，现在broker3找不到了，自然也无法找到group对应的GroupCoordinator。

根据三个服务的group id，计算对应的partition值，真相大白，broker3找不到是真凶。
 ```
 scala> Math.abs("defy-rtlog-receiver-cdn".hashCode()) % 50  // 正常启动
res8: Int = 33

scala> Math.abs("defy-rtlog-receiver-midsrc".hashCode()) % 50   // 无法启动
res9: Int = 8

scala> Math.abs("defy-rtlog-receiver-srcproxy".hashCode()) % 50  // 无法启动
res10: Int = 38
 ```

# 解决问题
启动一台kafka broker id为3的机器即可。
另外将特殊topic副本设为3，这样即使一台机器挂掉，`__consumer_offsets`上该partition的leader可以选举到其它机器上，不会产生上述影响
```
# server.properties 文件中
offsets.topic.replication.factor=3
```

# 进一步思考
* broker3关闭的时候，联系不到GroupCoordinator，consumer group消费的offset变动，kafka怎么记录呢？
* broker3启动的时候，上面保存的offsets信息必然不准甚至丢失，此时GroupCoordinator如何告诉consumer group消息消费到哪里了？

# kafka历史花絮
kafka 0.9.0版本之后，开始启用新的consumer config，采用bootstrap.servers替代之前版本的zookeeper.connect，弱化对zk的依赖，将对zk的依赖隐藏到broker后面。有如下两个相关改动：

* 在server端增加了GroupCoordinator这个角色
* 将topic的offset信息存储从zk改到一个特殊的topic中（`__consumer_offsets`）

# reference
[聊聊kafka中的group coordinator](https://www.jianshu.com/p/833b64e141f8)

[Kafka Detailed Consumer Coordinator Design](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design)

[Kafka源码解析之GroupCoordinator详解10](https://matt33.com/2018/01/28/server-group-coordinator/)
