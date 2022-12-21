### RocketMQ 部署架构

**RocketMQ的角色介绍**

- Producer：消息的发送者；举例：发信者
- Consumer：消息接受者；举例：收信者
- Broker：暂存和传输消息；举例：邮局
- NameServer：管理Broker;举例：各个邮局的管理机构
- Topic：区分消息的种类；一个发送者可以发送消息给一个或者多个Topic；一个消息的接收者可以订阅一个或者多个Topic消息
- Message Queue：相当于是Topic的分区；用于并行发送和接受请求

<img src="https://fenchuiyun-scw.oss-cn-shanghai.aliyuncs.com/blog/image-20221111102549230.png" alt="image-20221111102549230" style="zoom: 67%;" />

- NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步
- Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。注意：当前RocketMQ版本在部署架构上支持一Master多Slave，但只有BrokerId=1的从服务器才会参与消息的读负载。
- Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
- Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/0）,以及从服务器是否可读等因素建议下次是从Master还是Slave拉取。



执行流程：

1. 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。
2. Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。
3. 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送发送消息时自动创建Topic。
4. Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所有的Broker建立长连接从而向Broker发消息。
5. Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息



### RocketMQ特性

##### 订阅与发布

消息的发布是指某个生产者向某个topic发送消息；消息的订阅是指某个消费者关注了某个topic中带有某些tag的消息。

##### 消息顺序

消息有序指的是一类消息消费时，能按照发送的顺序来消费。例如：一个订单产生了三条消息分别是在订单创建、订单付款、订单完成。消费时要按照这个顺序消费才能有意义，但是同时订单之间时可以并行消费的。RocketMQ可以严格保证消息有序。

##### 消息过滤

RocketMQ的消费者可以根据Tag进行消息过滤，也支持自定义属性过滤。消息过滤目前时在Broker端实现的，有点是减少了对与Consumer无用消息的网络传输，缺点是增加了Broker的负担、而且实现相当复杂。

##### 消息可靠性

RocketMQ支持消息的高可靠，影响消息可靠性的几种情况：

1. Broker非正常关闭
2. Broker异常Crash
3. OS Crash
4. 机器掉电，但是能立即恢复供电情况
5. 机器无法开机（可能是cpu、主板、内存等关键设备损坏）
6. 磁盘设备损坏

1、2、3、4四种情况都属于硬件资源可立即恢复情况，RocketMQ在这四种情况下能保证消息不丢失，或者丢失少量数据（依赖刷盘方式是同步还是异步）

5、6属于单点故障，且无法回复，一旦发生，在此单点上的消息全部消失。

RocketMQ在这两种情况下，通过异步复制，可保证99%的消息不丢，但是仍然会有极少量的消息可能丢失。

通过同步双写技术可以完全避免单点，同步双写势必会影响性能，适合消息可靠性要求极高的场景，例如与Money相关的应用。

##### 至少一次

至少一次（At least Once）指每个消息必须被投递一次。Consumer先Pull消息到本地，消息消费完后，才向服务器返回ack，如果没有消费一定不会ack，所以RocketMQ可以很好的支持此特性。

##### 回溯消费

回溯消费是指Consumer已经消费成功的消息，由于业务上需求需要重新消费，要支持此功能，Broker在向Consumer投递成功消息后，消息仍然需要保留。并且重新消费一般是按照时间维度，例如由于Consumer系统故障，恢复后需要重新消费1小时前的数据，那么broker要提供一种机制，可以按照时间维度来回退消费进度。RocketMQ支持按照时间回溯消费，时间精确到毫秒。

##### 事务消息

RocketMQ事务消息（Transactional Message）是指应用本地事务和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败。

RocketMQ的事务消息提供类似X/Open XA的分布事务功能，通过事务消息能达到分布式事务的最终一致性。

##### 定时消息

定时消息（延迟队列）是指消息发送到broker后，不会立即被消费，等待特定时间投递给真正的topic。

broker有配置项messageDelayLevel，默认值为“1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h",18个level。

messageDelayLevel是broker的属性，不属于某个topic。发消息时，设置delayLevel等级即可：

msg.setDelayLevel(level)。level有以下三种情况:

- level==0，消息为非延迟消息
- 1<=level<=maxLevel，消息延迟特定时间，例如level==1，延迟1s
- level>maxLevel，则level==maxLevel，例如level==20，延迟2h

定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，queueId=delayTimeLevel - 1，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。Broker会调度消费SCHEDULE_TOPIC_XXXXX，将消息写入真实的topic。

##### 消息重试

Consumer消费失败后，要提供一种重试机制，令消息在消费一次。Consumer消费消息失败通常可以认为有以下几种情况：

1. 由于消息本身的原因，例如反序列化失败，消息数据本身无法处理（例如话费充值，当前消息的手机号被注销，无法充值）等。这种错误通常需要跳过这条消息，在消费其他消息，而这条失败的消息即使立刻重试消费，99%也不成功，所以最好提供一种定时重试机制，即过10秒后再重试。
2. 由于依赖的下游应用服务不可用，例如db连接不可用，外系统网络不可达等。遇到这种错误，即使跳过当前失败的消息，消费其他消息同时也会报错。这种情况建议应用sleep 30s，在消费吓一跳消息，这样可以减轻Broker重试消息的压力。

##### 消息重投

生产者在发送消息时：

- 同步消息失败会重投
- 异步消息又重试
- oneway没有任何保证

消息重投保证消息尽可能发送成功、不丢失，但可能会造成消息重复，消息重复在RocketMQ中是无法避免的问题。消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会大概率事件。另外，生产者主动重发、consumer负载变化也会导致重复消费。

如下方法可以设置消息重试策略：

1. retryTimesWhenSendFailed：同步发送失败重投次数，默认为2，因此生产者会最多尝试发送retryTimesWhenSendFailed+1次。不会选择上次失败的broker，尝试向其他broker发送，最大程度保证消息不丢失。超过重投次数，抛异常，由客户端保证消息不丢失。当出现RemotingException、MQClientException和部分MQBrokerException时会重投。
2. retryTimesWhenSendAsyncFailed：异步发送失败重试次数，异步重试不会选择其他broker，仅在同一个broker上做重试，不保证消息不丢。
3. retryAnotherBrokerWhenNotStoreOK：消息刷盘（主或备）超时或slave不可用（返回状态非SEND_OK），是否尝试发送到其他broker，默认false。十分重要消息可以开启。



##### 流量控制

生产者控流，因为broker处理能力达到瓶颈；消费者流控，因为消费能力达到瓶颈。

1. 生产者流控

   - commitLog文件被锁时间超过osPageCacheBusyTimeOutMills时，参数默认为1000ms，发生流控。
   - 如果开启transientStorePoolEnable=true，且broker为异步刷盘的主机，且transienStorePool中资源不足，拒绝当前send请求，发生流控。
   - broker每隔10ms检查send请求队列头部请求的等待时间，如果超过waitTimeMillsInSendQueue，默认200ms，拒绝当前send请求，发生流控。
   - broker通过拒绝send请求方式实现流量控制。

   注意，**生产者流控，不会尝试消息重投。**

2. 消费者流控

   - 消费者本地缓存消息数超过pullThreasholdForQueue时，默认1000。
   - 消费者本地缓存消息大小超过pullThesholdSizeForQueue时，默认100MB。
   - 消费者本地缓存消息跨度超过consumerConcurrentlyMaxSpan时，默认2000。
   - 消费者流控的结果是降低拉去频率。

##### 死信队列

死信队列用于处理无法被正常消费的消息。

当一条消息初次消费失败，消息队列会自动进行消息重试；

达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时消息队列不会立刻将消息丢失，而是将其发送到该消费者对应的特殊队列中。

RocketMQ将这种正常情况下无法被消费的消息称为死信队列（Dead-Letter Message）,在RocketMQ中，可以通过使用console控制台对死信队列中的消息进行重发来使得消费者实例再次进行消费。



### 消费模式Push or Pull

RocketMQ消费订阅有两种模式，一种是Push模式（MQPushConsumer），即MQServer主动向消费端推送；另一种是Pull模式（MQPullConsumer），即消费端在需要时，主动到MQServer拉取。但在具体实现时，Push和pull模式本质都是采用消费端主动拉取的方式，即consumer轮询从broker拉取消息。

- Push模式特点：

  好处就是实时性高。不好处在消费端的处理能力有限，当瞬间推送很多消息给消费端时，容易造成消费端的消息积压，严重时会压垮客户端。

- Pull模式

  好处就是主动权掌握在消费端自己手中，根据自己的处理能力量力而行。缺点就是如何控制Pull的频率。定时间隔太久担心影响时效性，间隔太短担心做太多“无用功”浪费资源。比较这种的办法就是长轮训。

- Push模式与Pull模式的区别

  Push方式里,consumer把长轮训的动作封装了，并注册MessageListener监听器，取到消息后，唤醒MessageListener的consumeMessage()来消费，对用户而言，感觉消息是被推送过来的。

  Pull方式里，取消息的过程需要用户自己主动调用，首先通过打算消费的Topic拿到MessageQueue的集合，遍历MessageQueue集合，然后针对每个messageQueue批量取消息，一次取完后，记录该队列下一次要取的开始Offset，知道取完了，再换另一个MessageQueue。

**RocketMQ使用长轮训机制来模拟Push效果，算是兼顾了二者的优点**

<br>









### RocketMQ中的角色及相关术语

1. 消息模型（Message Model）

   RocketMQ主要由Producer、Broker、Consumer三部分组成，其中Producer负责生产消息，Consumer负责消费消息，Broker负责存储消息。Broker在实际部署过程中对应一台服务器，每个Broker可以存储多个Topic的消息，每个Topic的消息也可以分片存储于不同的Broker。MessageQueue用于存储消息的物理地址，每个Topic中的消息地址存储于多个MessageQueue中。ConsumerGroup由多个consumer实例构成。

2. Producer

   消费生产者，负责产出消息，一般由业务系统负责产生消息。

3. Consumer

   消息消费者，负责消费消息，一般是后台系统负责异步消费。

4. PushConsumer

   Consumer消费的一种类型，该模式下Broker收到数据后会主动推给消费端。应用通常向consumer对象注册一个Listener接口，一旦收到消息，Consumer对象立刻回调Listener接口方法。该消费模式一般实时性高。

5. PullConsumer

   Consumer消费的一种类型，应用通常主动调用Consumer的拉消息方法从Broker服务器拉消息、主动权由应用控制。一旦获取了批量消息，应用就会启动消费过程。

6. ProducerGroup

   同一类Producer的集合，这类producer发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发生之后奔溃，则Broker服务器会联系同一生产组的其他生产者实例以提交或回溯消费。

7. ConsumerGroup

   同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。要注意的是，消费者组的消费实例**必须订阅完全相同的Topic**。RocketMQ支持两种消费模式：集群消费（Clustering）和广播消费（Broadcasting）。

8. Broker

   消息中转角色，负责存储消息，转发消息，一般也称为Server。在JMS规范中称为provider。

9. 广播消费

   一条消息被多个Consumer消费，即使这些Consumer属于同一个Consumer Group，消息也会被Consumer Group中每个Consumer 都消费一次，广播消费中的 Consumer Group 概念可以认为在消息划分方面无意义。

   在CORBA Notification 规范中，消费方式都属于广播消费。

   在JMS规范中，相当于 JMS Topic （puslish/subscribe）模式

10. 集群消费

   一个Consumer Group 中的Consumer 实例平均分摊消息消费。例如某给topic有9条消息，其中一个Consumer Group有3个实例（可能3个进程，或者3台机器），那举每个实例只消费其中3条消息。

11. 顺序消息

    消费消息的顺序要同发送消息的顺序一致，在RocketMQ中主要指的是局部顺序，即一类消息为满足顺序性，必须Producer单线程发送，且发送到同一个队列中，这样Consumer就可以按照Producer发送的顺序去消费消息。

12. 普通顺序消息

    顺序消费的一种，正常情况下可以保证完全的顺序消费，但是一旦发生通信异常，Broker重启，由于队列总数发生变化，哈希取模后定位的队列会变化，产生短暂的消息顺序不一致。如果业务能容忍在集群异常情况（如某个Broker宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。

13. 严格顺序消费

    顺序消费的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式Failover特性，即Broker集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前还未实现）

14. MessageQueue

    在RocketMQ中，所有消息队列都是持久化的，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长的，访问其中的存储单元使用**OffSet**来访问，offset为java **Long**类型，64位，理论上在100年内不会溢出，所以认为是长度无限，另外队列中只保存最近几天的数据，之前的数据会按照过期时间来删除。也可以认为Message Queue是一个长度无限的数组，offset 就是下标。

15. 标签（Tag）

    为消息设置的标志，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。标签能够有效地保持代码的清晰度和连贯性，并优化RocketMQ提供的查询系统。消费者可以根据Tag实现不同子主题的不同消费逻辑，实现更好的扩展性。

16. 
