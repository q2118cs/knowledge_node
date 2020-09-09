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