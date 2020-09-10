## 消费发送

发送消息的5个步骤：

1. 设置Producer的GroupName；
2. 设置InstanceName，当一个Jvm需要启动多个Producer的时候，通过设置不同的InstanceName来区分，不设置的话系统使用默认名称“DEFAULT”。
3. 设置发送失败重试次数；
4. 设置NameServer地址；
5. 组装消息并发送。

消息的返回状态有4种：

1.  **FLUSH_DISK_TIMEOUT**：Broker的刷盘策略被设置成SYNC_FLUSH时，没有在规定时间内完成刷盘。
2. **FLUSH_SLAVE_TIMEOUT**：在主备方式下，并且Broker被设置成SYNC_MASTER方式，没有在设定时间内完成主从同步。
3. **SLAVE_NOT_AVAILABLE**：在主备方式下，并且Broker被设置成SYNC_MASTER，但是没有找到被配置成Slave的Broker。
4. **SEND_OK**：发送成功

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