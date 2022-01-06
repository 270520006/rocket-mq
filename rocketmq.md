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

consumer与Name Server集群中的其中一个节点(随机选择)建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。
Consumer每隔30s从Name server获取topic的最新队列情况，这意味着Broker不可用时，Consumer最多最需要30s才能感知。
Consumer每隔30s (由ClientConfifig中heartbeatBrokerInterval决定）向所有关联的broker发送心跳,Broker每隔10s扫描所有存活的连接，若某个连接2分钟内没有发送心跳数据，则关闭连接;并向该Consumer Group的所有Consumer发出通知，Group内的Consumer重新分配队列，然后继续消费。当Consumer得到master宕机通知后，转向slave消费，slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是一旦master恢复，未同步过去的消息会被最终消费掉。
消费者对列是消费者连接之后(或者之前有连接过)才创建的。我们将原生的消费者标识由 (IP}@{消费者group}扩展为{IP}@{消费者groupHtopicHtag}，（例如xXX.XXX.XXX.xxx@mqtest_producer
group_2m2sTest_tag-zyk)。任何一个元素不同，都认为是不同的消费端，每个消费端会拥有一份自己消费对列（默认是broker对列数量*broker数量)。

## 关键特性及实现原理

### 顺序消费

消息有序指的是可以按照消息的发送顺序来消费。
例如:一笔订单产生了3条消息，分别是订单创建、订单付款、订单完成。消费时，必须按照顺序消费才有意义，与此同时多笔订单之间又是可以并行消费的。
例如生产者产生了2条消息∶M1、M2，要保证这两条消息的顺序，应该怎样做?可能脑中想到的是这样的:

#### 1、发送消息后接收确认消息

![image-20211118113412470](rocketmq/image-20211118113412470.png)

但是这个模型存在的问题是:如果M1和M2分别发送到两台Server上

> * 就不能保证M1先到达MQ集群
> * 也不能保证M1被先消费。

#### 2、FIFO（发送到同一server下）

​	换个角度看，如果M2先与M1到达MQ集群，甚至M2被消费后，M1才到达消费端，这时候消息就乱序了，说明以上模型是不能保证消息的顺序的。如何才能在MQ集群保证消息的顺序?一种简单的方式就是将M1、M2发送到同一个Server上:

![image-20211118113903006](rocketmq/image-20211118113903006.png)

根据先到达先被消费的原则，M1会先于M2被消费，这样就保证了消息的顺序。FIFO
但是这个模型也仅仅是在理论上可以保证消息的顺序，在实际场景中可能会遇到下面的问题:网络延迟问题。

![image-20211118162608340](rocketmq/image-20211118162608340.png)

​	只要将消息从一台服务器发往另一台服务器，就会存在网络延迟问题，如上图所示，如果发送M1耗时大于发送M2耗时，那么仍然存在可能M2被先消费，仍然不能保证消息的顺序，即使M1和M2同时到达消费端，由于不清楚消费端1和消费端2的负载情况，仍然可能出现M2先于M1被消费的情况。
​	那如何解决这个问题呢?

#### 3、消息未得到确认再次发送

​	将M1和M2发往同一个server，且发送M1后，需要消费端响应成功后才能发送M2。
​	但是这里又会存在另外的问题:如果M1被发送到消费端后，消费端1没有响应，那么是继续发送M2呢，还是重新发送M1?一般来说为了保证消息一定被消费，肯定会选择重发M1到另外一个消费端2,如下图,保证消息顺序的正确方式:

![image-20211118162908346](rocketmq/image-20211118162908346.png)

​	但是仍然会有问题:消费端1没有响应Server时，有两种情况，

> * 一种是M1确实没有到达(数据可能在网络传输中丢失)
>
> * 另一种是消费端已经消费M1并且已经发回响应消息，但是MQ Server没有收到。
>
>   如果是第二种情况，会导致M1被重复消费。

​	回过头来看消息顺序消费问题，严格的顺序消息非常容易理解，也可以通过文中所描述的方式来简化处理，总结起来，要实现严格的顺序消息,简单可行的办法就是:

>保证生产者-MQServer-消费者是”一对一对一”的关系。
>注意这样的设计方案问题:
>1.并行度会成为消息系统的瓶颈(吞吐量不够)
>2.产生更多的异常处理。比如︰只要消费端出现问题，就会导致整个处理流程阻塞，我们不得不花费更多的精力来解决阻塞的问题。

#### 4、Rocket MQ的局部有序性

我们最终的目标是要集群的高容错性和高吞吐量，这似乎是一对不可调和的矛盾，那么rocketmq是如何解决的呢?

>世界上解决一个计算机问题最简单的方法:“恰好”不需要解决它!—阿里资深技术专家沈询

有些问题，看起来很重要，但实际上我们可以通过合理的设计将问题分解来规避，如果硬要把时间花在解决问题本身，实际上不仅效率低下，而且也是一种浪费。从这个角度来看消息的顺序问题，可以得出两个结论:

>* 不关注乱序的应用大量存在
>
>* 队列无序并不意味着消息无序

所以从业务层面来保证消息的顺序，而不仅仅是依赖于消息系统，是不是我们更应该寻求的一种合理的方式?最后从源码角度分析	RocketMQ怎么实现发送顺序消息。
RocketMQ通过轮询所有队列的方式来确定消息被发送到哪一个队列（负载均衡策略）。

比如下面的,所以rocket mq也无法做到全局有序性，只能保证局部有序性。

```java
// RocketMQ通过MessageQueueSelector中实现的算法来确定消息发送到哪一个队列上
// RocketMQ默认提供了两种MessageQueueselector实现:随机/Hash
//当然你可以根据业务实现自己的MessageQueueselector来决定消息按照何种策略发送到消息队列中SendResult sendResult =   @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId);
```

在获取到路由信息以后，会根据MessageQueueSelector实现的算法来选择一个队列，同一个Orderld获取到的肯定是同一个队列。

```java
private SendResult send( ){//获取topic路由信息
TopicPublishInfo topicPublishInfo =
this.tryToFindTopicPublishInfo(msg. getTopic() );
if (topicPublishInfo != null && topicPublishInfo.ok( )){
    MessageQueue mq = null;
1/根据我们的算法，选择一个发送队列1/这里的arg = orderId
mq = selector.select(topicPublishinfo .getMessageQueueList( ), msg, arg);
    if(mq !=null){
return this.sendKernelimpl(msg,mq，communicationMode，sendcallback,timeout )
    }
}
                          };
```

### 消息重复

造成消息重复的根本原因是:网络不可达。

只要通过网络交换数据，就无法避免这个问题。所以解决这个问题的办法是绕过这个问题。那么问题就变成了:如果消费端收到两条一样的消息，应该怎样处理?

* 消费端处理消息的业务逻辑要保持幂等性。
  * 原理：只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样。
  * 解决方案：很明显应该在消费端实现，不属于消息系统要实现的功能。
* 保证每条数据都有唯一编号，且保证消息处理成功与去重表的日志同时出现。
  * 原理：利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不在处理这条消息。
  * 解决方案：由消息系统实现，也可以由业务端实现。正常情况下出现重复消息的概率其实很小，如果由消息系统来实现的话，肯定会对消息系统的吞吐量和高可用有影响，所以最好还是由业务端自己处理消息重复的问题，这也是RocketMQ不解决消息重复问题的原因。RocketMQ不保证消息不重复，如果你的业务系统需要保证严格的不重复消息,需要你自己在业务端去重。

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
rocketmqinc/rocketmq:4.4.0 \
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
rocketmqinc/rocketmq:4.4.0 \
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
--name rmqconsoleng2 \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.56.101:9876  
-Dcom.rocketmq.sendMessageWithVIPChannel=false 
-Duser.timezone='Asia/Shanghai'" \
-v /etc/localtime:/etc/localtime \
-p 8080:8080 \
styletang/rocketmq-console-ng
```

至此，rocket-mq搭建完成，踩了许多坑，真的烦，访问地址：http://192.168.56.101:8080/#/

## 整合Springboot和Rocket MQ

这个整合，做好准备，非常的坎坷，如果遇到了bug可以看一下我整合的bug集，没有包含的，就自求多福了：

### 生产者

* 整体结构

![image-20220106165644002](rocketmq/image-20220106165644002.png)

* 导入maven依赖：

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
            <scope>provided</scope>
        </dependency>
        <!--rocketmq-->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.1.1</version>
        </dependency>
    </dependencies>
```

* 写配置文件

```yml
server:
  port: 8082

rocketmq:
  name-server: 192.168.56.101:9876
  producer:
    group: my-producer-group

logging:
  file:
    path: /usr/log/mqproductservice/mqproductservice.log
  level:
    root: INFO
    com.anran.projectmanage.mapper: DEBUG
```

* 写接口

```java
/**
 * @author zsp
 * @version 1.0
 * @date 2022/1/6 14:17
 */
@RestController
@RequestMapping("/product")
public class ProductController {
    private static final Logger log = LoggerFactory.getLogger(ProductController.class);
    @Autowired
    RocketMQTemplate rocketMQTemplate;
    @GetMapping("/msg")
    public String getMsg(){
        try {
            rocketMQTemplate.convertAndSend("first-topic",
                    Thread.currentThread().getName()+"使用rocketmq生产了一条消息");
            log.info("发送成功");
            return "发送成功！";
        } catch (MessagingException e) {
            log.info("发送失败");
            return "发送失败"; }}
}
```

* 查看成功结果

![image-20220106170346736](rocketmq/image-20220106170346736.png)

### 消费者

整体结构：

![image-20220106170105374](rocketmq/image-20220106170105374.png)

* 依赖同生产者
* 配置文件

```yml
server:
  port: 8083

#rocketmq配置信息
rocketmq:
  #nameservice服务器地址（多个以英文逗号隔开）
  name-server:  192.168.56.101:9876
  #消费者配置
  consumer:
    #组名
    group: my-producer-group
    #监听主题
    topic: first-topic
    #tags（监听多个tag时使用 || 进行分割，如果监听所有使用*或者不填）

logging:
  file:
    path: /usr/log/mqconsumerservice/mqconsumerservice.log
  level:
    root: INFO
    com.anran.projectmanage.mapper: DEBUG
```

* 编写接口

```java
@Component
@RocketMQMessageListener(topic = "first-topic",consumerGroup = "my-consumer-group")
@Slf4j
public class Consumer implements RocketMQListener<String> {
    @Override
    public void onMessage(String message) {
        System.out.println(message);
    }
}
```

* 查看成功结果

![image-20220106170440681](rocketmq/image-20220106170440681.png)

## bug集合

这个bug非常，非常的多，我快踩坑踩吐了...

#### bug1：时间不同步

* 出现时间不同步bug，先到容器内使用date，到主机也用date查看
  * 如果不一致，则将主机时间文件挂载到容器内（上面的容器创建代码已经修正）
  * 如果一致，则将主机时间同步到虚拟机时间上（我使用的是VirtualBox）
    * 先到VirtualBox的根目录D:\Program Files\Oracle\VirtualBox下，然后cmd打开
    * 接下来按以下输入就好了

```shell
D:\VirtualBox>VBoxManage list vms
"centos7" {360c208d-fa7a-427d-8cf9-dca0672fd0b6}
"centos7_2" {037d138b-d4d9-4684-a033-64e07dc89c92}

D:\VirtualBox>vboxmanage setextradata "centos7" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" "1"
```

![image-20220106111353116](rocketmq/image-20220106111353116.png)

最终解决：

![image-20220106113052603](rocketmq/image-20220106113052603.png)

#### bug2：系统内存不足

报错类型：

```shell
service not available now, maybe disk full, CL: 0.95 CQ: 0.95 INDEX: 0.95, maybe your broker mach
```

报错原因：rocket mq默认存储盘占用超75%会出错

解决办法：调整这个比例到99，先到rocketmq的broker中

```shell
[rocketmq@e54d0f365667 bin]$ pwd  
/opt/rocketmq-4.4.0/bin
[rocketmq@e54d0f365667 bin]$ vi runbroker.sh
#在这个文件里面加上即可
JAVA_OPT="${JAVA_OPT} -Drocketmq.broker.diskSpaceWarningLevelRatio=0.99"
```

## Rocket MQ源码

### 消息发送源码



