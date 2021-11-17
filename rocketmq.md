# RocketMQ

## MQ背景和选型

消息队列作为高并发系统的核心组件之一，能够帮助业务系统解构提升开发效率和系统稳定性。主要具有以下优势:

* 削峰填谷（主要解决瞬时写压力大于应用服务能力导致消息丢失、系统奔溃等问题)
* 系统解耦(解决不同重要程度、不同能力级别系统之间依赖导致一死全死)
  提升性能（当存在一对多调用时，可以发一条消息给消息系统，让消息系统通知相关系统)
* 蓄流压测（线上有些链路不好压测，可以通过堆积一定量消息再放开来压测)

目前主流的MQ主要是Rocketmq、kafka、Rabbitmq，Rocketmq相比于Rabbitmq、kafka具有主要优势特性有:

* 支持事务型消息（消息发送和DB操作保持两方的最终一致性,rabbitmq和kafka不支持)
* 支持结合rocketmq的多个系统之间数据最终一致性(多方事务，二方事务是前提)
* 支持延迟消息（rabbitmq和kafka不支持)
* 支持指定次数和时间间隔的失败消息重发（kafka不支持, rabbitmq需要手动确认)
* 支持consumer端tag过滤，减少不必要的网络传输(rabbitmq和kafka不支持)
* 支持重复消费(rabbitmq不支持,kafka支持)

## 集群部署

![image-20211109173819998](rocketmq/image-20211109173819998.png)

### Name Server

Name Server是一个几乎无状态节点，可集群部署，**节点之间无任何信息同步**。类似kafka的zk，但是zk是同步的。

### Broker

Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的Broker id，不同的Broker ld来定义，Brokerld为0表示Master，非0表示Slave。

每个Broker与Name Server集群中的所有节点建立长连接，定时(每隔30s)注册Topic信息到所有NameServer。Name Server定时(每隔10s)扫描所有存活broker的连接，如果Name Server超过2分钟没有收到心跳，则Name Server断开与Broker的连接。

注：和zk不同，Brocker会连接Name Server的集群上的每一个机子，将元数据（数据备份）存到上面。

### Producer 

Producer与Name Server集群中的其中一个节点(随机选择)建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
Producer每隔30s (由ClientConfifig的pollNameServerInterval)从Name server获取所有topic队列的最新情况，这意味着如果Broker不可用，Producer最多30s能够感知，在此期间内发往Broker的所有消息都会失败。
producer每隔30s(由ClientConfifig中heartbeatBrokerInterval决定)向所有关联的broker发送心跳，Broker每隔10s中扫描所有存活的连接，如果Broker在2分钟内没有收到心跳数据，则关闭与Producer的连接。

### Consumer

lonsumer与Name Server集群中的其中一个节点(随机选择)建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。
Consumer每隔30s从Name server获取topic的最新队列情况，这意味着Broker不可用时，Consumer最多最需要30s才能感知。
Consumer每隔30s (由ClientConfifig中heartbeatBrokerInterval决定）向所有关联的broker发送心跳,Broker每隔10s扫描所有存活的连接，若某个连接2分钟内没有发送心跳数据，则关闭连接;并向该Consumer Group的所有Consumer发出通知，Group内的Consumer重新分配队列，然后继续消费。当Consumer得到master宕机通知后，转向slave消费，slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是一旦master恢复，未同步过去的消息会被最终消费掉。
消费者对列是消费者连接之后(或者之前有连接过)才创建的。我们将原生的消费者标识由 (IP}@{消费者group}扩展为{IP}@{消费者groupHtopicHtag}，（例如xXX.XXX.XXX.xxx@mqtest_producer
group_2m2sTest_tag-zyk)。任何一个元素不同，都认为是不同的消费端，每个消费端会拥有一份自己消费对列（默认是broker对列数量*broker数量)。



## 安装rocket mq

### 创建nameserver服务

* 拉取镜像

```shell
docker pull rocketmqinc/rocketmq
```

* 构建容器

```shell
docker run -d \
--restart=always \
--name rmqnamesrv \
-p 9876:9876 \
-v /home/rocketmq/data/namesrv/logs:/root/logs \
-v /home/rocketmq/data/namesrv/store:/root/store \
-e "MAX_POSSIBLE_HEAP=100000000" \
rocketmqinc/rocketmq \
sh mqnamesrv 
```

>-d	以守护进程的方式启动
>- -restart=always	docker重启时候容器自动重启
>- -name rmqnamesrv	把容器的名字设置为rmqnamesrv
>- -p 9876 : 9876	把容器内的端口9876挂载到宿主机9876上面
>- -v /docker/rocketmq/data/namesrv/logs:/root/logs	把容器内的/root/logs日志目录挂载到宿主机的 /docker/rocketmq/data/namesrv/logs目录
>- -v /docker/rocketmq/data/namesrv/store:/root/store	把容器内的/root/store数据存储目录挂载到宿主机的 /docker/rocketmq/data/namesrv目录
>- rmqnamesrv	容器的名字
>- -e “MAX_POSSIBLE_HEAP=100000000”	设置容器的最大堆内存为100000000
>- rocketmqinc/rocketmq	使用的镜像名称
>- sh mqnamesrv	启动namesrv服务

### 创建broke结点

* 创建配置文件

```shell
vim /home/rocketmq/conf/broker.conf
# 所属集群名称，如果节点较多可以配置多个
brokerClusterName = DefaultCluster
#broker名称，master和slave使用相同的名称，表明他们的主从关系
brokerName = broker-a
#0表示Master，大于0表示不同的slave
brokerId = 0
#表示几点做消息删除动作，默认是凌晨4点
deleteWhen = 04
#在磁盘上保留消息的时长，单位是小时
fileReservedTime = 48
#有三个值：SYNC_MASTER，ASYNC_MASTER，SLAVE；同步和异步表示Master和Slave之间同步数据的机制；
brokerRole = ASYNC_MASTER
#刷盘策略，取值为：ASYNC_FLUSH，SYNC_FLUSH表示同步刷盘和异步刷盘；SYNC_FLUSH消息写入磁盘后才返回成功状态，ASYNC_FLUSH不需要；
flushDiskType = ASYNC_FLUSH
# 设置broker节点所在服务器的ip地址
brokerIP1 = 192.168.52.136  # 注意：改成你的IP地址

```

* 构建brocker结点的docker容器

```shell
docker run -d  \
--restart=always \
--name rmqbroker \
--link rmqnamesrv:namesrv \
-p 10911:10911 \
-p 10909:10909 \
-v  /home/rocketmq/data/broker/logs:/root/logs \
-v  /home/rocketmq/data/broker/store:/root/store \
-v /home/rocketmq/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf \
-e "NAMESRV_ADDR=namesrv:9876" \
-e "MAX_POSSIBLE_HEAP=200000000" \
rocketmqinc/rocketmq \
sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf
```

>参数	说明
>
>* -d	以守护进程的方式启动
>* –restart=always	docker重启时候镜像自动重启
>
>- -name rmqbroker	把容器的名字设置为rmqbroker
>- --link rmqnamesrv : namesrv	和rmqnamesrv容器通信
>- -p 10911 :10911	把容器的非vip通道端口挂载到宿主机
>- -p 10909 : 10909	把容器的vip通道端口挂载到宿主机
>- -e “NAMESRV_ADDR= namesrv:9876”	指定namesrv的地址为本机namesrv的ip地址:9876
>- -e “MAX_POSSIBLE_HEAP=200000000”	rocketmqinc/rocketmq sh mqbroker 指定broker服务的最大堆内存
>- rocketmqinc/rocketmq	使用的镜像名称
>- sh mqbroker -c /docker/rocketmq/conf/broker.conf	指定配置文件启动broker节点，该配置文件对应上面 vim 编辑的配置文件

### 创建rockermq-console服务

```shell
docker run -d \
--restart=always \
--name rmqadmin \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.56.101:9876 \
-Dcom.rocketmq.sendMessageWithVIPChannel=false" \
-p 9999:8080 \
pangliang/rocketmq-console-ng
```

至此，rocket-mq搭建完成，踩了许多坑，真的烦。访问地址：http://192.168.56.101:9999/#/
