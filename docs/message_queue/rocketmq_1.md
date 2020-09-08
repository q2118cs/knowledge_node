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









\#!/bin/sh 

\# Licensed to the Apache Software Foundation (ASF) under one or more 

\# contributor license agreements. See the NOTICE file distributed with 

\# this work for additional information regarding copyright ownership. 

\# The ASF licenses this file to You under the Apache License, Version 2.0 

\# (the "License"); you may not use this file except in compliance with 

\# the License. You may obtain a copy of the License at 

\#

\# http://www.apache.org/licenses/LICENSE-2.0 

\#

\# Unless required by applicable law or agreed to in writing, software 

\# distributed under the License is distributed on an "AS IS" BASIS, 

\# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 

\# See the License for the specific language governing permissions and 

\# limitations under the License. 

\#========================================================================== 

================= 

\# Java Environment Setting 

\#========================================================================== 

================= 

error_exit () 

broker:

echo "ERROR: $1 !!" 

exit 1 

}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java 

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java 

[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME 

variable in your environment, We need java(x64)!" 

export JAVA_HOME 

export JAVA="$JAVA_HOME/bin/java" 

export BASE_DIR=$(dirname $0)/.. 

export 

CLASSPATH=.:${BASE_DIR}/conf:${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib/* 

\#========================================================================== 

================= 

\# JVM Configuration 

\#========================================================================== 

================= 

JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m - 

XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=160m" 

JAVA_OPT="${JAVA_OPT} -XX:CMSInitiatingOccupancyFraction=70 - 

XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 - 

XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8" 

JAVA_OPT="${JAVA_OPT} -verbose:gc -Xlog:gc:/dev/shm/rmq_srv_gc.log - 

XX:+PrintGCDetails" 

JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow" 

JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages" 

\# JAVA_OPT="${JAVA_OPT} - 

Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib" 

\#JAVA_OPT="${JAVA_OPT} -Xdebug - 

Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n" 

JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}" 

JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}" 

$JAVA ${JAVA_OPT} $@

