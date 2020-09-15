## RocketMQ部署架构

### 角色及相关术语

- Producer：消息的生产者；与NameServer随机的一个节点建立长连接，定期从NameServer获取Topic信息，与NameServer的master建立长连接，定时向NameSever发送心跳。Producer完全无状态，可集群部署。
- Consumer：消息消费者；与NameServer随机的一个节点建立长连接，Consumer既可以从master订阅消息，也可以从slave订阅消息。
- Broker：暂存和传输消息。Broker分为master和slave，master和slave对应关系通过指定相同的BrokerName，master和slave的BrokerId不同，master的BrokerId一般为0，非0的表示slave，只有BrokerId为1的才参与消息的读负载，其他的slave复制同步数据。每个Broker与NameServer建立长连接，定时注册Topic信息到所有NameServer。
- NameServer：管理Broker，一个无状态的节点，可集群部署，节点间无信息同步。
- Topic：消息主题；消息生产者可以发送消息给一个或多个topic，消息消费者可以订阅一个或多个topic消息。
- Message Queue：相当于topic的分区，用于并行发送和接收消息。
- 广播消费：一条消息被多个consumer消费，即使Consumer 属于同一个consumer group，消息也会在每个consumer中消费一次。
- 集群消费：一个consumer group中的consumer实例平均分摊消息。
- 顺序消息：消费顺序和发送顺序一致，主要指的局部顺序，即一类消费顺序一致。
  - 普通顺序消息：正常情况可以保证完全顺序消费，但是在集群异常情况下会产生短暂的顺序不一致。如果业务能容忍消息短暂乱序，推荐使用此方式。
  - 严格顺序消费：无论正常还是异常都能保证消息的顺序，但是会牺牲Failover特性，比如集群中一个Broker不可用则整个集群都不可用。
- 标签(Tag)：为消息设置的标志，区分同一topic下不同消息类型，标签能保证代码清晰度和连贯性，消费者可根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性

RocketMQ基于订阅发布机制，一个Topic有多个队列，一个Broker为每个主题创建4个读4个写队列，多个Broker组成一个集群，集群有相同的多台Broker组成master-slave架构，brokerId为0代表master，大于0的为slave。

### 特性

- 订阅与发布：生产者向某个topic发送消息，消费者监听某个topic中带某些tag的消息。
- 消息顺序：无法保证整个消息队列有序，除非只使用一个mq。
- 消息可靠性：软件异常，资源可立即恢复的情况下，RocketMQ通过刷盘方式是异步还是同步来保证消息不丢失或只丢失部分消息；硬件故障的话，当前节点数据全部丢失，这种情况可以主从来解决，并开启同步双写，但同步双写会影响性能，适合对消息可靠性要求极高的场景。RocketMQ从3.0开始支持同步双写。
- 回溯消费：支持重复消费，RocketMQ支持按时间回溯。
- 事务消息：支持分布式事务功能。
- 定时消息：RocketMQ支持延迟队列，Broker支持messageDelayLevel属性，可通过此参数设置延迟，该参数属于Broker不属于topic。定时消息会存在SCHEDUAL_TOPIC_XXX的topic中，根据delayTimeLevel存入特定的queue，queueId=delayTimeLevel-1，即每个queue存的消息延迟属性是一致的，Broker会调度消费该topic将消息存入真实的topic。
- 消息重试：同步发送消息会重试，通过retryTimesWhenSendFailed控制；异步消息有重试，通过retryTimesWhenSendAsyncFailed控制；消息刷盘超时或slave不可用，可通过retryAnotherBrokerWhenNotStoreOK开启重试。
- 流量控制：支持生产者和消费者流控，生产者流控不会尝试消息重投。
- 死信队列：处理无法被正常消费的消息。

### 消费模式

RocketMQ支持Push和Pull模式，但在具体实现的时候本质上都是采用consumer轮训从broker拉取消息，通过长轮训的方式来模拟push的效果。

## 消费发送

发送消息的5个步骤：

1. 设置Producer的GroupName；
2. 设置InstanceName，当一个Jvm需要启动多个Producer的时候，通过设置不同的InstanceName来区分，不设置的话系统使用默认名称“DEFAULT”。
3. 设置发送失败重试次数；
4. 设置NameServer地址；
5. 组装消息并发送。

消息的返回状态有4种：

1.  **FLUSH_DISK_TIMEOUT**：Broker的刷盘策略被设置成SYNC_FLUSH时，没有在规定时间内完成刷盘。
2.  **FLUSH_SLAVE_TIMEOUT**：在主备方式下，并且Broker被设置成SYNC_MASTER方式，没有在设定时间内完成主从同步。
3.  **SLAVE_NOT_AVAILABLE**：在主备方式下，并且Broker被设置成SYNC_MASTER，但是没有找到被配置成Slave的Broker。
4.  **SEND_OK**：发送成功

提升写入性能的方式：

1. 在一些对速度要求高，但是可靠性要求不高的场景下，比如日志收集类应用， 可以采用Oneway方式发送，Oneway方式只发送请求不等待应答，即将数据写入客户端的Socket缓冲区就返回，不等待对方返回结果，用这种方式发送消息的耗时可以缩短到微秒级。
2. 增加Producer的并发量，使用多个Producer同时发送，producer是线程安全的，可以多线程使用。

## 消息消费

提高consumer处理速度的方式：

1. 同一个consumer group下增加consumer实例，但是consumer数量不要超过Topic下Read Queue数量，超过的不会消费；
2. 提高单个consumer并行处理线程数，修改consumeThreadMin和consumeThreadMax；
3. 以批量方式消费，设置consumer的consumeMessageBatchMaxSize这个参数，如果设置为N，在消息多的时候每次收到的是个长度为N的消息链表；
4. 检测延时情况，跳过非重要消息。

## 消息存储

mq是通过刷盘的方式来持久化消息的，消息的顺序写速度可以达到600mb/s，随机写速度100kb/s，所以RocketMQ使用顺序写，保证了消息存储速度。

RocketMQ消息存储主要由三个文件组成：

1. **CommitLog**：消息主体以及元数据的存储主体，单个文件默认1G，文件名为偏移量，有20位，比如00000000001073741824代表0-1073741824偏移量的消息。
2. **ConsumeQueue**：消费消息队列，主要为了提高消费的性能，consume可以根据consumequeue来查找消息，相当于消费消息的索引，里面主要保存了三种数据：commitLog中的偏移量、消息大小、消息tag的hash值。
3. **IndexFile**：提供了一种可以通过key或时间区间来查询消息的方法，底层使用hashmap结构，因此rocketmq索引底层实现为hash索引。

## 消息过滤

由于consumer端订阅消息是通过consumeQueue拿到索引然后从commitLog获取消息，所以rocketMQ是通过consumer端订阅消息时做消息过滤的，主要支持两种过滤方式：

1. **tag过滤**：多个tag可用||分隔，服务端根据tag的hashcode过滤，消费端获取到消息后还要根据原始tag字符串比较，如果不同则丢弃该消息。
2. **SQL92过滤**：只对push的消息起作用，通过sql表达式过滤，为了提高效率，mq使用了布隆过滤器避免了每次都执行。需要在broker.conf开启sql92支持

## 零拷贝

mmap：将一个文件或者其它对象映射进内存，应用程序内存和内核缓冲区内存已经映射好了，直接磁盘读取数据到内核缓冲区，由于已经和应用程序产生了映射，不需要再读取到应用程序内存，然后将内核缓冲区数据写到scoket缓冲区，再发送到网络。适合小文件发送。

sendfile：直接向内核空间发送请求，内核会从磁盘读取相应内容到内核缓冲区，内核缓冲区映射到scoket缓冲区再向外发送，不需要用户态到内存态的映射。适合发送大文件。

之所以叫零拷贝，是因为数据在内存中没有发生拷贝，只是在内存和I/O设备间传输。

rocketmq使用mmap实现零拷贝，kafka使用sendfile实现的零拷贝。由于kafka的应用场景偏向于大数据，rocketmq应用场景偏向于业务，因此实现方式不同。

## 同步和异步复制

1. **同步复制**：等broker的master和slave都写成功后返回给客户端写成功状态。会增大数据写入延迟，降低吞吐量
2. **异步复制**：只要broker的master写入成功就返回给客户端写成功状态。但会造成数据丢失。

通过broker.conf文件中的brokerRole配置，有三种配置：

1. SYNC_MASTER：当前broker是同步复制的master；
2. ASYNC_MASTER：当前broker是异步复制的master；
3. SLAVE：当前broker是个slave。

## 高可用

- 消费高可用：consumer可以不指定从master还是从slave读取消息，当master不可用的时候会自动切换到slave。
- 发送高可用：可以将topic的多个队列创建在多个broker组上，这样当一个broker组不可用的时候可以从别的broker组发送消息。

## 刷盘机制

rocketMQ持久化消息的时候都是先写入PageCache，然后刷盘，可以保证内存和磁盘都有一份数据。刷盘分为同步和异步两种方式。

二者的主要区别在于：异步刷盘写完PageCache直接返回，同步刷盘需要等待刷盘完成才返回。

## 消息重试

1. 顺序消息：为了保证消息的顺序应用会出现消息消费被阻塞的情况，每隔1s会进行重试。

2. 无序消息：只针对集群消息有效，广播方式不提供重试机制，默认每条消息最多重试16次，重试配置方式：

   - 返回 ConsumeConcurrentlyStatus.RECONSUME_LATER; （推荐）
   - 返回null
   - 抛异常

   如果想要不重试，则要捕获异常并返回ConsumeConcurrentlyStatus.CONSUME_SUCCESS。

## 死信队列

RocketMQ中消息重试超过一定次数后（默认16次）就会被放到死信队列中，死信队列不会被消费者再消费，有效期与正常消息相同为3天。一个死信队列对应一个groupId，并且包含了该groupId下所有的死信消息，不论消息属于哪个topic。

## 延迟消息

 broker有配置项messageDelayLevel，默认值为“1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h”，18个level。定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，queueId=delayTimeLevel – 1，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。

## 顺序消息

顺序消息分为全局顺序消息和部分顺序消息：全局顺序消息需要topic的读写队列都为1，producer和consumer并发数也都为1，为了保证有序性，只能单线程消费；一般使用的是部分有序，Consumer使用MessageListenerOrderly保证收到的消息是有序的

## 事务消息

RocketMQ可以做分布式事务，基于两阶段提交的方式，其执行步骤如下：

- producer向mq发送一条“待确认”的消息；
- mq将“待确认”消息持久化，向producer回复消息已经发送成功；
- producer执行本地事务；
- producer根据本地事务结果向mq发送commit还是rollback消息，如果是commit则mq将消息设置为可投递的属性，否则删除掉消息；如果mq一直没收到消息则会自动回查producer查看消息状态，默认回查15次。

备注：由于mq是顺序写消息的，因此消息是不能被删除的，因此mq引入了Op消息，如果一条事务消息没有对应的Op消息，说明这个

事务的状态还无法确定，事务commit还是rollback都会记录一个Op消息


